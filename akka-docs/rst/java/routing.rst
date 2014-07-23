
.. _routing-java:

路由
=======

消息可通过一个路由器来发送，从而高效地将其路由到目标actor，称为 *routees*. 一个 ``Router`` 可以再actor内部或外部使用，并且你可以自己管理routee，或者使用具有配置功能的自包含的路由actor。

可以根据你的应用程序需求使用不同的路由策略。Akka自带了一些开箱即用的有用的路由策略。但是，如你在本章将会看到的，还可以 :ref:`创建自己的 <custom-router-java>`.

.. _simple-router-java:

一个简单的路由器
^^^^^^^^^^^^^^^

下面的例子阐述了如何使用一个 ``Router`` 并且在actor内部管理routee。

.. includecode:: code/docs/jrouting/RouterDocTest.java#router-in-actor

我们创建一个 ``Router`` 并且制定了它在将消息路由到routee时应当使用 ``RoundRobinRoutingLogic`` 。

Akka所自带的路由逻辑有：

* ``akka.routing.RoundRobinRoutingLogic``
* ``akka.routing.RandomRoutingLogic``
* ``akka.routing.SmallestMailboxRoutingLogic``
* ``akka.routing.BroadcastRoutingLogic``
* ``akka.routing.ScatterGatherFirstCompletedRoutingLogic``
* ``akka.routing.ConsistentHashingRoutingLogic``

我们将这些routee创建为包装在 ``ActorRefRoutee`` 中的普通子actor。我们监控这些routee，从而能够在它们终止时能够替换它们。

通过路由器发送消息是通过 ``route`` 方法来进行的, 如上面的例子中对 ``Work`` 消息所做的操作。

``Router`` 是不可变的，而 ``RoutingLogic`` 是线程安全的; 这意味着它们还可以在actor外部使用。

.. note::

	大体上，任何发送到路由器的消息会被继续发送到它的（一个）routee，但是有一个例外。
	特殊的 :ref:`broadcast-messages-java` 会发送到路由器的 *全部* routee。 

路由Actor
^^^^^^^^^^^^^^

路由器还可以被创建为一个自包含的actor，自己管理routee，并且通过配置加载路由逻辑以及其他设置项。

路由actor的类型具有两种不同的风格：

* Pool - 这个路由器将routee创建为子actor，并且在其终止时从路由器中移除。
  
* Group - routee Actor在路由器外部被创建，并且路由器使用actor selection来将消息发送到指定的路径，不监控终止。

路由器actor的设置可以再配置中定义，也可以编程指定。
尽管路由actor可以再配置文件中定义，但它们仍然必须通过编程方式来创建。也就是，你不可以单独通过外部配置来创建一个路由器。如果你在配置文件中定义了路由actor，那么这些设置将取代任何编程指定的参数而被使用。

你将消息通过路由actor发送给routee的方式跟常规actor相同，也就是通过其 ``ActorRef``。 路由器actor将消息转发到routee，不修改原始的发送者。当一个routee回复一个被路由的消息时，这个回复消息会被发送到原始的发送者，而不是路由actor。

.. note::

	大体上，任何被发送到路由器的消息会被发送到其routee，但是有些例外。这些例外在下面的 :ref:`router-special-messages-java` 部分中进行了说明。

Pool
----

下列代码和配置片段展示了如何创建一个 :ref:`round-robin
<round-robin-router-java>` 路由器，它将消息转发到 五个 ``Worker`` 路由器。 routee会被创建为路由器的子actor。

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-round-robin-pool

.. includecode:: code/docs/jrouting/RouterDocTest.java#round-robin-pool-1

这是相同的例子，但是路由器配置通过编程方式而不是配置来指定。

.. includecode:: code/docs/jrouting/RouterDocTest.java#round-robin-pool-2

R远程部署的 Routee
***********************

除了能够创建本地actor作为routee，你还可以命令路由器将其所创建的子actor部署到一组远程主机上。Routee将按照一种 round-robin 风格部署。为了远程部署actor，应当将路由配置包装到一个 ``RemoteRouterConfig`` 中，附上要部署到的节点的远程地址。 远程部署要求在claspath中包含 ``akka-remote`` 模块。

.. includecode:: code/docs/jrouting/RouterDocTest.java#remoteRoutees

发送者
*******


当一个routee发送一条消息时，它可以 :ref:`将自己设置为发送者<actors-tell-sender-java>`.

.. includecode:: code/docs/jrouting/RouterDocTest.java#reply-with-self

然后，将*路由器*设置为发送者通常是有用的。例如，当你想将routee的细节隐藏在路由器背后时，你可能想要将路由器设置为发送者。下面的代码片段展示了如何将父路由器设置为发送者。

.. includecode:: code/docs/jrouting/RouterDocTest.java#reply-with-parent


监管
***********

由pool路由器创建的routee就被创建为路由器的子actor。路由器因此也是子actor的监管者。

路由器actor的监管策略可以通过Pool的 ``supervisorStrategy`` 属性进行配置。如果未提供配置，则路由器默认使用一个 “总是上报” 的策略。这意味着错误会被发送到路由器的监管者以进行处理。路由器的监管者将决定如何处理错误。

注意路由器的监管者会将这个错误当做路由器自己的错误。因此一个停止或重启指令将会导致路由器 *自身* 停止或重启。路由器进而会导致其子actor停止和重启。

还有一点应当提到，就是路由器的重启行为已经被覆盖为，当重启时，在仍会重建子actor的同时，将仍然在pool中保留相同数目的actor。

这意味着，如果你没有指定路由器或其父actor的 :meth:`supervisorStrategy` ，则一个routee中的错误会上报到路由器的父actor，这在默认情况下会重启路由器，这将重启所有的routee（它使用Escalete，并且在重启过程中不停止routee）。原因是为了让默认的行为下，向子actor的定义中添加 :meth:`.withRouter` 不会影响应用到子actor的监管策略。这种低效的情况可以通过在定义路由器时指定路由策略来避免。

设置策略很容易做到:

.. includecode:: code/docs/jrouting/RouterDocTest.java#supervision

.. _note-router-terminated-children-java:

.. note::

  如果一个pool路由器的子actor终止，pool路由器不会自动创建一个新的子actor。在一个pool路由器的所有子actor都终止的情况下，路由器会终止自身，除非它是一个动态路由器，例如，使用一个resizer。

Group
-----

有时，所要的行为并不是让路由actor创建其routee，而是单独创建routee，并且将其提供给路由器来使用。你可以通过将routee的路径传入路由器的配置而实现。消息会通过 ``ActorSelection`` 发送到这些路径。

下面的例子显示了如何通过提供三个routee actor的路径字符串来创建一个路由器actor。

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-round-robin-group

.. includecode:: code/docs/jrouting/RouterDocTest.java#round-robin-group-1

这是同一个例子，但是其中路由器配置通过编程而不是配置来指定。
.. includecode:: code/docs/jrouting/RouterDocTest.java#round-robin-group-2

routee actor在路由器外部创建：

.. includecode:: code/docs/jrouting/RouterDocTest.java#create-workers

.. includecode:: code/docs/jrouting/RouterDocTest.java#create-worker-actors

对于运行于远程主机上的actor，这些路径可以包含协议和地址信息。远程调用要求 ``akka-remote`` 模块被包含于 classpath。

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-remote-round-robin-group

路由器用法
^^^^^^^^^^^^
在本届中我们会描述如何创建不同类型的路由器actor。

本节中的路由actor创建自一个顶层的actor，名为 ``parent`` 。注意配置中的部署路径以  ``/parent/`` 开头，其后是路由器actor的名称。

.. includecode:: code/docs/jrouting/RouterDocTest.java#create-parent

.. _round-robin-router-java:

RoundRobinPool 和 RoundRobinGroup
----------------------------------

按照一种 `round-robin <http://en.wikipedia.org/wiki/Round-robin>`_ 风格来路由到它的 routee.

RoundRobinPool 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-round-robin-pool

.. includecode:: code/docs/jrouting/RouterDocTest.java#round-robin-pool-1

RoundRobinPool 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java#round-robin-pool-2

RoundRobinGroup 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-round-robin-group

.. includecode:: code/docs/jrouting/RouterDocTest.java#round-robin-group-1

RoundRobinGroup 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java
   :include: paths,round-robin-group-2

RandomPool and RandomGroup
--------------------------

这种路由器类型为每条消息随机地选择一个routee
This router type selects one of its routees randomly for each message.

RandomPool 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-random-pool

.. includecode:: code/docs/jrouting/RouterDocTest.java#random-pool-1

RandomPool 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java#random-pool-2

RandomGroup 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-random-group

.. includecode:: code/docs/jrouting/RouterDocTest.java#random-group-1

RandomGroup 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java
   :include: paths,random-group-2

.. _balancing-pool-java:

BalancingPool
-------------

一种会试图从忙碌的routee向闲置的routee重新分配工作的路由器。
所有routee共享同一个信箱。

BalancingPool 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-balancing-pool

.. includecode:: code/docs/jrouting/RouterDocTest.java#balancing-pool-1

BalancingPool 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java#balancing-pool-2

针对pool所使用的负载均衡dispatcher的额外配置，可以被配置到路由器部署配置中的 ``pool-dispatcher`` 一节。

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-balancing-pool2

BalancingPool 没有Group变种

SmallestMailboxPool
-------------------

一种试图向信箱中消息最少的非挂起子routee发送消息的路由器。
选择按照一下这种顺序进行：

 * 挑选任意具有空信箱的闲置routee（没有在处理消息）
 * 挑选任意具有空信箱的routee
 * 挑选信箱中待处理消息最少的routee
 * 挑选任一远程routee，远程actor被认为具有最低优先级，因为它们的信箱大小是未知的

SmallestMailboxPool 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-smallest-mailbox-pool

.. includecode:: code/docs/jrouting/RouterDocTest.java#smallest-mailbox-pool-1

SmallestMailboxPool 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java#smallest-mailbox-pool-2

SmallestMailboxPool 不存在Group变种，因为信箱的大小和内部调度状态从routee的路径并不可知。

BroadcastPool 和 BroadcastGroup 
--------------------------------

一个广播路由器会将它所接收的消息转发给所有 *routee* 

BroadcastPool 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-broadcast-pool

.. includecode:: code/docs/jrouting/RouterDocTest.java#broadcast-pool-1

BroadcastPool 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java#broadcast-pool-2

BroadcastGroup 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-broadcast-group

.. includecode:: code/docs/jrouting/RouterDocTest.java#broadcast-group-1

BroadcastGroup 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java
   :include: paths,broadcast-group-2

.. note::

  广播路由器总是将 *每个* 消息转发到其routee。如果你不想广播所有消息，那么你可以使用一个非广播路由器，并且在需要的适合使用 :ref:`broadcast-messages-java` 。


ScatterGatherFirstCompletedPool 和 ScatterGatherFirstCompletedGroup
--------------------------------------------------------------------

ScatterGatherFirstCompletedRouter 会将消息发送到其所有routee。它然后会等待收到第一个回复。这个结果然后会发回原始发送者。其他回复会被丢弃。

在所配置的时间之内，可以期待至少一个回复，否则它将回复一个包含 ``akka.pattern.AskTimeoutException`` 的 ``akka.actor.Status.Failure`` 。

ScatterGatherFirstCompletedPool 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-scatter-gather-pool

.. includecode:: code/docs/jrouting/RouterDocTest.java#scatter-gather-pool-1

ScatterGatherFirstCompletedPool 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java#scatter-gather-pool-2

ScatterGatherFirstCompletedGroup 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-scatter-gather-group

.. includecode:: code/docs/jrouting/RouterDocTest.java#scatter-gather-group-1

ScatterGatherFirstCompletedGroup 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java
   :include: paths,scatter-gather-group-2

ConsistentHashingPool 和 ConsistentHashingGroup
------------------------------------------------

The ConsistentHashingPool uses `一致性hash <http://en.wikipedia.org/wiki/Consistent_hashing>`_
来基于所发送的消息选择一个routee。 这篇 
`文章 <http://weblogs.java.net/blog/tomwhite/archive/2007/11/consistent_hash.html>`_ 给出了关于一致性hash如何实现的优秀见解。

有3中方式可以指定将哪些数据用于一致性hash的键。

* 你可以定义路由器的 ``withHashMapper`` ，将到来的消息映射到它们的hash键。这使得这一决定对发送者透明。

* 这些消息可以实现 ``akka.routing.ConsistentHashingRouter.ConsistentHashable`` 。这个键是消息的一部分，并且将其同消息一同定义式非常方便的。
 
* 消息可被包装于一个 ``akka.routing.ConsistentHashingRouter.ConsistentHashableEnvelope``
，定义将哪些数据用于一致性hash键。发送者了解索要使用的键。
 
 定义一致性hash键的三种方式可以一起使用，并且同时对于一个actor， ``withHashMapper`` 首先被尝试。


代码示例:

.. includecode:: code/docs/jrouting/ConsistentHashingRouterDocTest.java#cache-actor

.. includecode:: code/docs/jrouting/ConsistentHashingRouterDocTest.java#consistent-hashing-router

在上例中，你可以看到 ``Get`` 消息自身实现了 ``ConsistentHashable`` , 同时 ``Entry`` 消息被包装在一个 ``ConsistentHashableEnvelope`` 中。 ``Evict`` 消息由 ``hashMapping`` partial function.

ConsistentHashingPool 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-consistent-hashing-pool

.. includecode:: code/docs/jrouting/RouterDocTest.java#consistent-hashing-pool-1

ConsistentHashingPool 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java#consistent-hashing-pool-2

ConsistentHashingGroup 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-consistent-hashing-group

.. includecode:: code/docs/jrouting/RouterDocTest.java#consistent-hashing-group-1

ConsistentHashingGroup 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java
   :include: paths,consistent-hashing-group-2


``virtual-nodes-factor`` 是用于一致性hash节点环中每个routee的虚拟节点个数，使分发更加一致。

.. _router-special-messages-java:

特殊地处理消息
^^^^^^^^^^^^^^^^^^^^^^^^^^

多数发送到路由actor的消息会根据路由器的路由逻辑而被转发。然而有少数几类消息具有特殊行为。
注意这些特殊消息，除了 ``Broadcast`` 消息, 只被自包含的路由actor处理，而不是 :ref:`simple-router-java` 中所描述的 ``akka.routing.Router`` 组件。

.. _broadcast-messages-java:

Broadcast消息
------------------

一个 ``Broadcast`` 消息可以被用于向一个路由器的 *所有* routee发送一条消息。 当一个路由器收到一条 ``Broadcast`` 消息时，它会将那条消息的 *载荷* 广播到所有的routee，无论那个路由器正常情况下如何路由它的消息。

下面的例子展示了如何使用一个 ``Broadcast`` 消息来讲一个非常重要的消息发送到路由器的每个routee。

.. includecode:: code/docs/jrouting/RouterDocTest.java#broadcastDavyJonesWarning

在这个例子中，路由器收到 ``Broadcast`` 消息，提取其载荷
(``"Watch out for Davy Jones' locker"``), 然后将载荷发送到路由器所有的routee。每个routee actor负责处理所接收到的载荷消息。

PoisonPill 消息
-------------------

``PoisonPill`` 对于所有actor都具有特殊的处理方式，包括路由器。 当任一actor接收到一条 ``PoisonPill`` 消息时，那个actor会停止。参见 :ref:`poison-pill-java` 文档了解细节。

.. includecode:: code/docs/jrouting/RouterDocTest.java#poisonPill

对于一个路由器，它通常会讲消息传递给routee，很重要的是要意识到 ``PoisonPill`` 消息仅被路由器处理。发送到一个路由器的 ``PoisonPill`` *不*会被继续发给 routee.

然而，一条发送给路由器的 ``PoisonPill`` 消息仍然会影响其routee，因为它会停止路由器，当路由器停止时它还会停止其子actor。停止子actor是正常的actor行为。路由器将停止作为其子actor创建的routee。每个子actor将处理当前消息，然后停止。这可能导致一些消息不被处理。参见 :ref:`stopping-actors-java` 文档了解更多信息。

如果你希望停止一个路由器及其routee，但是你想让routee首先处理信箱中当前所有消息，那么你不应当向路由器发送一个 ``PoisonPill`` 消息。你应当将 ``PoisonPill`` 消息包装到一个 ``Broadcast`` 消息中，这样每个routee会收到这条 ``PoisonPill`` 消息。注意这将停止所有routee，也急速hi，甚至是编程式提供给路由器的routee。

.. includecode:: code/docs/jrouting/RouterDocTest.java#broadcastPoisonPill

使用上面所示的代码，每个routee会收到一条 ``PoisonPill`` 消息。每个routee 会继续照常处理它的消息，最终处理 ``PoisonPill`` 。这将导致routee停止。 在所有routee停止之后，路由器自身也会被 :ref:`自动停止 <note-router-terminated-children-java>` ，除非他是一个动态路由器，例如使用一个resizer。

.. note::

  Brendan W McAdams' 精彩的博客文章 `Distributing Akka Workloads - And Shutting Down Afterwards
  <http://blog.evilmonkeylabs.com/2013/01/17/Distributing_Akka_Workloads_And_Shutting_Down_After/>`_
  更加详细地讨论了 ``PoisonPill`` 消息能如何用于关闭路由器和routee。

Kill 消息
-------------

``Kill`` 消息是另一类被特殊处理的消息。参见 :ref:`killing-actors-java` 以了解actor如何处理 ``Kill`` 消息的概要信息。

当一个 ``Kill`` 消息被发送给一个路由器时，路由器会内部处理这条消息， 并且 *不* 将此消息继续发送给routee。路由器会抛出一个 ``ActorKilledException`` 并失败。然后它要么会被继续，要么被重启或终止，取决于它如何被监管。

作为路由器子actor的Routee还会被挂起，并且会被应用到路由器的监管指令所影响。不是路由器子actor的routee，也就是在路由器外部创建的actor，不会受到影响。

.. includecode:: code/docs/jrouting/RouterDocTest.java#kill

跟 ``PoisonPill`` 消息一样, 杀死一个router并间接杀死其子actor（刚好是其routee），与直接杀死routee（其中某些可能不是子actor） 是有区别的。要直接杀死routee，路由器应当将 ``Kill`` 消息包装在一个 ``Broadcast`` 消息之中发送。

.. includecode:: code/docs/jrouting/RouterDocTest.java#broadcastKill

管理消息
---------------------

* 向一个路由器actor发送一条 ``akka.routing.GetRoutees`` 会使之发回它当前所使用的routee，包装在一条 ``akka.routing.Routees`` 消息中。
* 向一个路由器actor发送一条 ``akka.routing.AddRoutee`` 会将相应的routee添加到路由器的routee集合中.
* 向一个路由器actor发送一条 ``akka.routing.RemoveRoutee`` 会将相应的routee从路由器的routee集合中删除。
* 向一个pool
路由器actor发送一条 ``akka.routing.AdjustPoolSize`` 消息会在路由器的routee集合中添加或删除相应数目的routee。

这些管理消息可能在其他消息之后被处理，因此如果在一个普通消息之后立即发送一条 ``AddRoutee`` ，并不能保证这条普通消息所被路由到的routee会改变。 如果你需要知道这一改变何时生效，你可以紧跟着 ``AddRoutee`` 发送一条 ``GetRoutees`` ，并且当你接收到 ``Routees`` 回复时，你知道之前的修改已经生效。

.. _resizable-routers-java:

动态调整大小的Pool
^^^^^^^^^^^^^^^^^^^^^^^^^^

所有pool都可以使用固定数目的routee或一种调整大小的策略以动态调整routee数量。

带有resizer的Pool 在配置中定义:

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-resize-pool

.. includecode:: code/docs/jrouting/RouterDocTest.java#resize-pool-1

更多的可用配置选项在 :ref:`configuration` 中进行了描述。

带有resizer的Pool 在代码中定义:

.. includecode:: code/docs/jrouting/RouterDocTest.java#resize-pool-2

*还值得注意的是，如果你在配置文件中定义 ``router`` ，那么这个值将被使用，而不是编程式发送的参数*

.. note::

  调整大小通过向actor pool发送消息来触发，但是它去完全不是同步的；相反，一个消息被发送到 “head” ``RouterActor`` 来执行大小变更。 这样你就不能依赖调整大小而在其他worker都忙碌的情况下立即创建新的worker。因为刚发送的消息会被排队到一个忙碌actor的信箱。为了纠正这种情况，配置pool让它使用一个负载均衡dispatcher，参见 `配置 Dispatcher`_ 了解更多信息。

.. _router-design-java:

在 Akka 内部路由如何实现
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在表面上，路由器跟普通的actor看起来很像，但是它们的实现方式事实上不同。路由器被设计得及其高效地接收消息并迅速将它们发送给routee。

普通的actor可被用来路由消息，但是一个actor的单线程处理可能成为瓶颈。 路由器可以对通常的消息处理流水线进行优化，允许并行路由，来大幅提高吞吐量。这是通过将路由逻辑嵌入路由actor的 ``ActorRef`` 而不是路由actor本身来实现的。发送到一个路由器的 ``ActorRef`` 的消息可以立即被路由到routee， 彻底跳过了单线程的路由器actor。

当然，这样做的代价是，路由代码的内部比使用常规actor来实现更加复杂。幸运的是所有这些复杂性对于路由API的消费者并不可见。 然后，当实现自己的路由器时，这是需要意识到的。

.. _custom-router-java:

自定义路由器
^^^^^^^^^^^^^
如果你没有找到任何能够满足需求的Akka提供的actor，那么你可以创建自己的路由器。为了让你的路由器能够运行，你需要满足特定的标准，这些标准在本节中将进行解释。

在创建自己的路由器之前，你应当考虑一个具有类似于路由器的行为的常规acto是否能跟一个成熟的路由器一样好地完成工作。如 :ref:`上 <router-design-java>` 所解释, 路由器跟常规actor相比的主要好处是它们更高的性能。但是它们比常规actor稍微更加难以编写。因此如果在你的应用程序中，较低的最大吞吐量是可以接受的，那么你可能会坚持使用传统actor。然而这一节假定你希望得到最大的吞吐量，因此阐述了如何创建自己的路由器。

这个例子中创建的路由器将每个消息复制到若干目标。

从路由逻辑开始：

.. includecode:: code/docs/jrouting/CustomRouterDocTest.java#routing-logic

``select`` 将为每个消息而被调用，并且在此例中通过round-robin挑选一些目标，重用了已有的 ``RoundRobinRoutingLogic`` 并且将结果包装在一个 ``SeveralRoutees`` 实例中.  ``SeveralRoutees`` 会将这条消息发动给提供的所有routee。

这一路由逻辑的实现必须是线程安全的，因为它可能在actor外部使用。

路由逻辑的一个单测: 

.. includecode:: code/docs/jrouting/CustomRouterDocTest.java#unit-test-logic

你应当在此停止，并且如 :ref:`simple-router-java`  所描述的那样，将 ``RedundancyRoutingLogic`` 与``akka.routing.Router`` 一同使用。

让我们继续，并且将此变为一个自包含的，可配置的，路由器actor。

创建一个继承 ``PoolBase``, ``GroupBase`` 或 ``CustomRouterConfig`` 的类. 那个类是一个路由逻辑的工厂，并且持有路由器的配置。在这里我们使用一个 ``Group``.

.. includecode:: code/docs/jrouting/RedundancyGroup.java#group

这个路由器的用法与Akka所提供的路由器完全相同。
This can be used exactly as the router actors provided by Akka.

.. includecode:: code/docs/jrouting/CustomRouterDocTest.java#usage-1

注意我们在 ``RedundancyGroup`` 中添加了一个需要一个 ``Config`` 参数的构造器，这使得能够在配置中定义它。

.. includecode:: ../scala/code/docs/routing/CustomRouterDocSpec.scala#jconfig

注意 ``router`` 属性中的全限定类名。路由类必须继承 ``akka.routing.RouterConfig`` (``Pool``, ``Group`` or ``CustomRouterConfig``) 并且包含具有一个 ``com.typesafe.config.Config`` 参数的构造器。
配置中的部署段会被传入构造器。

.. includecode:: code/docs/jrouting/CustomRouterDocTest.java#usage-2
 
配置 Dispatcher
^^^^^^^^^^^^^^^^^^^^^^^
用于创建pool子actor的dispatcher将从 ``Props`` 获取，如 :ref:`dispatchers-scala` 所述。

为了使得定义pool的routee的dispatcher变得容易，你可以在配置中的部署段中以inline方式定义dispatcher，

.. includecode:: ../scala/code/docs/routing/RouterDocSpec.scala#config-pool-dispatcher

只是你为一个pool启用一个专用的dispatcher所唯一需要做的事情。

.. note::

   如果你使用一组actor并且路由到它们的路径，那么它们仍将使用它们的 ``Props``中所配置的相同dispatcher，在actor创建之后不可能改变它的dispatcher。

“head” 路由器不可能一直运行于相同的dispatcher上，因为它处理的并不是相同类型的消息，因此这种特殊的actor并不使用 ``Props`` 中配置的dispatcher，而是从 :class:`RouterConfig` 中获取 ``routerDispatcher`` , 这默认为actor系统的默认dispatcher。所有标准路由器允许在它们的构造器或工厂方法中设置此属性，自定义路由器必须以何时的方式实现这个方法。

.. includecode:: code/docs/jrouting/RouterDocTest.java#dispatchers

.. note::
   将 ``routerDispatcher`` 配置为一个 :class:`akka.dispatch.BalancingDispatcherConfigurator` 是不被允许的，因为针对特殊的路由actor的消息不能被其他actor处理。
 