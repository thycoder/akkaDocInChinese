.. _fault-tolerance-java:

容错
======================

正如 :ref:`actor-systems` 中所解释的，每个actor都是其子actor的监管者，并且正因此，每个actor都定义了错误处理监管策略。这一策略不能后续修改，因为它使actor系统结构中不可分割的一部分。

错误处理实战
--------------------------

首先，让我们查看一个示例，它阐释了处理数据存储错误的一种方式，这是真实世界的应用程序中一种典型的错误源。当然当数据存储不可用时，能做什么取决于具体的应用，但是在这个例子中我们使用了尽可能重新连接的方案。

阅读以下的源代码。行内注释解释了错误处理的不同部分，以及它们为何被添加。运行这个示例也是非常值得推荐的，因为跟随日志输出来理解运行时发生了什么，是非常容易的。

.. toctree::

   fault-tolerance-sample

创建一个监管策略
------------------------------

下述小节详细更深入地解释了错误处理及其替代方案。

为了示范，我们考虑如下的策略：

.. includecode:: code/docs/actor/FaultHandlingTest.java
   :include: strategy

我选择了一些常见的异常类型用于示范 :ref:`supervision` 中所描述的错误处理指令。从第一个开始，它是一个one-for-one策略，意思是每个子actor被分别处理( all-for-one 策略的工作方式非常类似，唯一的区别是任何决定都会被应用到监管者所有的子actor上面，而不仅仅是出错的那个)。重启频率设置了范围，也就是每分钟最多10次。 ``-1`` 和 ``Duration.Inf()`` 意味着相应的范围不做限制，留下了为重启指定一个绝对上界以及让重启无限进行的可能性。当超过范围时，子actor会被停止。

.. note::

  如果所声明的策略是在监控actor的内部声明（而不是一个单独的类），那么其决策函数能够以线程安全的方式访问actor所有内部状态，包括获得当前失败的子actor的引用(可通过错误消息的 ``getSender`` 获取).

默认监管策略
^^^^^^^^^^^^^^^^^^^^^^^^^^^

如果所定义的策略没有涵盖所抛出的异常，则 ``Escalate`` 会被使用。

如果一个actor的监管策略没有定义，那么以下异常会默认被处理：

* ``ActorInitializationException`` 将停止出错的子actor
* ``ActorKilledException`` 将停止出错的子actor
* ``Exception`` 将重启出错的子actor
* 其他类型的 ``Throwable`` 将被升级到父actor

如果异常一路升级，直到根守卫actor，它会按照与上述定义相同的默认策略来进行处理。

停止监管策略
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

更接近于Erlang的方式是这样的策略：仅当子actor出错时停止它们，，然后当DeathWatch发出子节点丢失的信号时，在监管者中采取正确的动作。 这种策略也已经打包提供为 :obj:`SupervisorStrategy.stoppingStrategy` 以及伴随它的 :class:`StoppingSupervisorStrategy` 配置器，当你想让``"/user"`` 守卫actor应用它时，需要使用。

记录actor故障
^^^^^^^^^^^^^^^^^^^^^^^^^

默认情况下 ``SupervisorStrategy`` 仅当故障被升级时才回记录。升级的故障被认为应当在更高层次上进行处理，并可能会进行记录。

你可通过在实例化``SupervisorStrategy``时将 ``loggingEnabled`` 设置为 ``false`` 来关闭默认日志。 自定义的日志可以在 ``Decider`` 中进行。注意如果 ``SupervisorStrategy`` 实在监管actor内部申明的，则当前失败的actor的引用可通过 ``getSender`` 来访问。

你还可以在你自己的 ``SupervisorStrategy`` 实现中通过覆盖 ``logFailure`` 方法来定制日志。

监管顶层 Actor
-------------------------------

顶层actor的意思是那些使用 ``system.actorOf()`` 创建的actor， 并且它们是 :ref:`User Guardian <user-guardian>` 的子actor。 在这种情况下没有特殊规则，守卫actor仅仅遵循所配置的策略。

测试应用程序
----------------

下一节展示了实际应用中不同指令的效果， 为此需要进行测试安装。首先我们需要一个适当的监管者：

.. includecode:: code/docs/actor/FaultHandlingTest.java
   :include: supervisor

这个监管者将被用来创建一个子actor，供我们做实验：

.. includecode:: code/docs/actor/FaultHandlingTest.java
   :include: child

使用 :ref:`akka-testkit` 中所描述的工具，测试变得更简单, 其中 ``TestProbe`` 提供了一个actor引用，用于接收和检查回复。

.. includecode:: code/docs/actor/FaultHandlingTest.java
   :include: testkit

让我们创建一些actor

.. includecode:: code/docs/actor/FaultHandlingTest.java
   :include: create

第一个测试应当示范 ``Resume`` 指令，因此我们通过在actor中设置一些非初始状态并让它出错，来试试看:

.. includecode:: code/docs/actor/FaultHandlingTest.java
   :include: resume

如你可见，值42躲过了错误处理指令。现在，如果我们将错误修改为更加严肃的 ``NullPointerException``, 情况就不再是这样了:

.. includecode:: code/docs/actor/FaultHandlingTest.java
   :include: restart

最终在致命的 ``IllegalArgumentException`` 的情况下，子actor会被监管者终止:

.. includecode:: code/docs/actor/FaultHandlingTest.java
   :include: stop

到现在为止，监管者彻底没有受到子actor出错的影响，因为指令集合确实处理了错误。在 ``Exception`` 的情况下, 这不再成立，监管者升级了这个错误。

.. includecode:: code/docs/actor/FaultHandlingTest.java
   :include: escalate-kill

监管者自身由 :class:`ActorSystem` 所提供的顶级actor监管，其默认策略师在所有的 ``Exception`` 情况下重启 (除了
``ActorInitializationException`` 和 ``ActorKilledException`` 这两个明显的例外). 因为重启情况下默认的指令是杀死所有子actor, 我们不期待可怜的子actor能在这次错误中存活。

为了避免这不是所需要的行为（取决于使用情况），我们需要使用不同的监管者，覆盖这种行为。

.. includecode:: code/docs/actor/FaultHandlingTest.java
   :include: supervisor2

有了这个父actor，子actor在升级的重启中幸存，如最后一个测试所示：

.. includecode:: code/docs/actor/FaultHandlingTest.java
   :include: escalate-restart

