.. _untyped-actors-java:

################
 Actors
################

`Actor Model`_ 为编写并发和分布式系统提供了更高的抽象层次。它减轻了开发人员显式地处理锁和线程管理的职责，使编写并发和并行系统变得更加容易。Actor模型是由Carl Hewitt在1973年的论文中所定义的，但由于Erlang语言而变得流行起来，一个在爱立信取得巨大成功的使用案例是，构建了高度并发且可靠的电信系统。

Akka的Actor API类似于Scala的Actor，但从Erlang借用了一些语法。

.. _Actor Model: http://en.wikipedia.org/wiki/Actor_model


创建Actor
===============

.. note::
 
 由于Akka采用强制性的父子监管，每一个actor都会被监管，并且（可能会）是子actor的监管者；我们建议你熟悉一下 :ref:`actor-systems` 和 :ref:`supervision` ，并且阅读 :ref:`addressing` 也可能会有所帮助。

定义一个Actor class
-----------------------

Java中的Actor通过继承 ``UntypedActor`` 类来实现，并且实现
:meth:`onReceive` 方法。 这个方法将消息作为一个参数。

这里是一个例子:

.. includecode:: code/docs/actor/MyUntypedActor.java#my-untyped-actor

Props
-----

:class:`Props` 是一个用于为actor的创建指定选项的配置类，可将其看成一个不可变类，因此是一个可以自由共享的用于创建一个actor的菜谱，包含相关的部署信息（例如，使用哪个调度器，更多内容参见后续内容）。这里是关于如何创建一个 :class:`Props` 实例的例子.

.. includecode:: code/docs/actor/UntypedActorDocTest.java#import-props
.. includecode:: code/docs/actor/UntypedActorDocTest.java#creating-props-config

第二行代码展示了如何将构造器参数传递给正在被创建的 :class:`Actor` 。匹配的构造器的存在，是在构造 :class:`Props` 对象的过程中进行验证的, 如果0个或多个匹配的构造器被找到，则会导致一个 :class:`IllegalArgumentEception` 。

第三行阐述了如何使用 :class:`Creator<T extends Actor>` 。创建者类必须是静态的，这是在 :class:`Props` 构造过程过程中进行验证的。类型参数的上界被用于却递给你所生成的actor的类如果被完全擦除则退化为 :class:`Actor` 。一个参数化工厂的例子是:

.. includecode:: code/docs/actor/UntypedActorDocTest.java#parametric-creator

.. note::

  为了满足让信箱的需求-类似于是为使用stash的actor配备一个基于deque的信箱-能够得到满足，actor类型必须在创建之前已知，这正是 :class:`Creator` 的类型参数所允许的。因此请确保尽可能地是为你的actor使用具体类型。

推荐的最佳实践 ^^^^^^^^^^^^^^^^^^^^^

有个好的想法是在 :class:`UntypedActor` 上提供静态方法，帮助保持 :class:`Props` 的创建尽可能地接近于actor的定义。 这还允许使用基于 :class:`Creator` 的方法，静态地验证所使用的构造器确实存在，而不必依赖于运行时检查。

.. includecode:: code/docs/actor/UntypedActorDocTest.java#props-factory

使用Props创建Actor
--------------------------

Actor是通过传入一个 :class:`Props` 实例到 :class:`ActorSystem` 或
:class:`ActorContext` 的工厂方法 :meth:`actorOf` 而创建的。

.. includecode:: code/docs/actor/UntypedActorDocTest.java#import-actorRef
.. includecode:: code/docs/actor/UntypedActorDocTest.java#system-actorOf

使用 :class:`ActorSystem` 会创建顶层actor，由actor系统所提供的守卫actor监管，而使用actor的context将创建一个子actor。

.. includecode:: code/docs/actor/UntypedActorDocTest.java#context-actorOf
   :exclude: plus-some-behavior

推荐创建一个包含子actor，孙actor，等等的层次结构，使之匹配应用程序的错误处理逻辑结构，参见 :ref:`actor-systems` 。

调用 :meth:`actorOf` 会返回 :class:`ActorRef` 的一个实例。 这是一个指向actor实例的句柄，是与actor交互的唯一方式。 :class:`ActorRef` 是不可变的，并且跟其所表示的actor是一一对应的关系。 :class:`ActorRef` 还是可序列化和可感知网络的。这意味着你可以将其序列化，将其发送到网络上，并且在远程主机上使用它，而它仍然会跨越网络表示原始节点上相同的actor。

名称参数是可选的，但是命名你的actor是更好的，因为名字会被用在日志消息中，并且用于标识actor。 名称必须非空，且不能以 ``$`` 开头, 但它可以包含进行了URL编码的字符(例如 ``%20`` 表示空格).  如果给定的名称已被相同父actor之下的其他子actor占用，则会抛出一个 `InvalidActorNameException` 。

Actor在创建时会自动地异步启动。

.. _actor-create-factory:

依赖注入
--------------------

如果你的UntypedActor具有一个含参构造器，那么这些参数也需要是 :class:`Props` 的一部分, 如 `above`__ 所述. 但是有些情况下应当使用工厂方法，例如当实际的构造器参数由一个依赖注入框架决定时。

__ Props_

.. includecode:: code/docs/actor/UntypedActorDocTest.java#import-indirect
.. includecode:: code/docs/actor/UntypedActorDocTest.java
   :include: creating-indirectly
   :exclude: obtain-fresh-Actor-instance-from-DI-framework

.. warning::

  你有时可能会倾向于提供一个 :class:`IndirectActorProducer` 它总是会返回相同的实例，例如，通过使用一个静态字段。这是不被支持的，因为它违背了actor重启的含义，它在此处进行了描述： :ref:`supervision-restart`.
  当使用一个依赖注入框架时，actor的bean *必须不* 具有单例作用域。

依赖注入和集成依赖注入框架的技术在以下网页有更深入的介绍： `Using Akka with Dependency Injection <http://letitcrash.com/post/55958814293/akka-dependency-injection>`_ 指导方针和Typesafe Activator中的 `Akka Java Spring <http://www.typesafe.com/activator/template/akka-java-spring>`_ 教程。

Inbox
---------

当在actor之外编写要跟actor通信的代码时，``ask`` 模式可以作为一个解决方案（参见后续内容），但是有两件事情是它做不到的： 接收多个回复（例如，通过让一个 :class:`ActorRef` 订阅一个通知服务）并且观察其它actor的生命周期。 出于这样的目的，有一个 :class:`Inbox` 类:

.. includecode:: code/docs/actor/InboxDocTest.java#inbox

:meth:`send` 方法会包装一个标准的 :meth:`tell` 并将内部actor的引用作为sender提供。这允许回复在最后一行被接收。观察一个actor也非常简单:

.. includecode:: code/docs/actor/InboxDocTest.java#watch

UntypedActor API
================

:class:`UntypedActor` 类只定义了一个抽象方法，就是上面提到的 :meth:`onReceive(Object message)`,  用来实现actor的行为。

如果当前actor的行为不能匹配一个收到的消息，那么推荐你调用 :meth:`unhandled`方法, 它默认会将一个 ``new akka.actor.UnhandledMessage(message, sender, recipient)`` 发布到actor系统的事件流(将配置项 ``akka.actor.debug.unhandled`` 设置为 ``on``来将它们转换为实际的Debug消息)。

另外，它还提供:

* :meth:`getSelf()` 引用actor的 :class:`ActorRef` 

* :meth:`getSender()` 引用最近收到的一条消息的发送方Actor, 典型的用法在 :ref:`UntypedActor.Reply` 中进行了描述

* :meth:`supervisorStrategy()` 用户可重写它来定义对子actor的监管策略
  这个策略典型地在actor内部声明，目的是为了在决策函数内能够访问actor的内部状态：因为故障作为一个消息被发送到监管者，并且跟其他消息一样处理（尽管在正常行为之外），actor内所有值和变量都是可用的， ``getSender()`` 也一样（其值将会是报告故障的中间子actor；如果原始的故障发生在一个遥远的后代actor之中，它仍然会被逐级上报）。

* :meth:`getContext()` 暴露了actor和当前的消息的上下文信息，例如:

  * 用于创建子actor的工厂方法 (:meth:`actorOf`)
  * actor所属的actor系统
  * 父监管者
  * 所监管的子actor
  * 生命周期监控
  * 热插拔的行为堆栈，在 :ref:`UntypedActor.HotSwap` 中进行了描述

剩余的可见方法都是用户可覆写的生命周期钩子方法，如下所述：

.. includecode:: code/docs/actor/UntypedActorDocTest.java#lifecycle-callbacks

上面所示的代码是由 :class:`UntypedActor` 类所提供的默认实现。

.. _actor-lifecycle-java:

Actor生命周期
---------------

.. image:: ../images/actor_lifecycle.png
   :align: center
   :width: 680

actor系统中的一个路径代表一个可以被活着的actor占据的地方。一个路径最初（除了系统初始化的actor之外）是空的。当 ``actorOf()`` 被调用时，它会将由传入的 ``Props`` 所描述的actor的一个 *化身* 分配到给定的路径。 一个actor化身由路径 *和一个UID* 来标识。 一次重启只会换掉由 ``Props`` 所定义的 ``Actor`` 实例，但是化身以及UID保持不变。

一个化身的生命周期在actor停止时结束。 在那时，适当的生命周期事件会被调用，并且监控者actor会被通知这一终止事件。在化身停止之后，路径可以再次复用，通过使用 ``actorOf()`` 创建一个actor。 在这种情况下，新化身的名称将与之前一样，但是UID会不同。

一个 ``ActorRef`` 总是表示一个化身 (路径 和 UID) 而不仅是给定的路径. 因此如果一个actor被停止，并且一个同名的新actor被创建，那么旧化身的 ``ActorRef`` 不会指向新的化身。

另一方面， ``ActorSelection`` 指向一个路径 (或多个路径，如果使用通配符的话) 而且彻底不知道哪个化身正在占据它。 ``ActorSelection`` 因此不能被监控。一种解析居住在一个路径下的当前化身的 ``ActorRef`` 的可能方法是通过发送一条 ``Identify`` 消息到 ``ActorSelection`` 这将收到一条 ``ActorIdentity`` 回复， 包含正确的引用(参见 :ref:`actorSelection-java`)。 这还可以使用 :class:`ActorSelection` 的 ``resolveOne`` 方法来实现, 它将返回一个所匹配的 :class:`ActorRef` 的一个 ``Future`` 对象。

.. _deathwatch-java:

生命周期监控，即DeathWatch
-----------------------------------

为了在其它actor终止 (也就是永久停止, 而不是临时的故障和重启)时收到通知, actor可以将自己注册为其它actor在终止时所发布的 :class:`Terminated` 消息 的接收者 (见 `停止Actor`_ )。 这个服务是由actor系统的 :class:`DeathWatch` 组件提供的。

注册一个监控者很简单（见第4行，其余部分用于说明整个的功能）：

.. includecode:: code/docs/actor/UntypedActorDocTest.java#import-terminated
.. includecode:: code/docs/actor/UntypedActorDocTest.java#watch

要注意 :class:`Terminated` 消息的产生与注册和终止行为所发生的顺序无关。特别地，即使在注册时被监控的actor已经终止，监控者actor仍然会受到一个 :class:`Terminated` 消息。

多次注册并不一定会导致多个消息产生，但是不能保证正好只有一个这样的消息被接收到：如果终止被监控的actor的动作已经生成了Terminated消息并且已经将其入队，并且在这个消息被处理之前又发生了另一次注册，则会有第二个消息进入队列，因为监控一个已经终止的actor的注册，会立刻导致 :class:`Terminated` 消息的产生。

还有可能使用 ``getContext().unwatch(target)`` 来停止监控另一个actor的死活。即使当 :class:`Terminated` 已经入队，这也是可行的； 在调用 :meth:`unwatch` 之后，另一个actor的任何 :class:`Terminated` 消息都将不再被处理。

启动钩子
----------

actor启动后，它的 :meth:`preStart` 方法会立即执行。

.. includecode:: code/docs/actor/UntypedActorDocTest.java#preStart

这一方法在actor第一次被创建时被调用。在重启过程中，它会被 :meth:`postRestart` 的默认实现所调用，这意味着通过重写那个方法，你可以选择这个方法中的初始化代码对于这个actor恰好仅执行一次，还是每次重启都执行。actor构造器中的初始化代码在actor类的实例被创建时总是会被调用，这在每次重启时都会发生。

重启钩子
-------------

所有的Actor都是被监管的， 也就是，使用某种故障处理策略与另一个actor连接在一起。 一旦在处理一个消息的时候抛出异常，Actor可能会被重启。这个重启过程涉及上面提到的钩子:

1. 旧的actor会以导致重启的异常和触发那条异常的消息作为参数，调用 :meth:`preRestart` ；如果重启并不是因为消息处理而导致的，则后者为 ``None`` , 例如，当一个监管者没有捕获某个异常，继而被它自己的监管者重启时，或者一个actor由于兄弟节点的故障而被重启时。如果消息可用，则其发送者也可以按照通常的方式访问（即通过调用 ``getSender()`` ）。 这个方法是完成清理、准备交接给新的actor实例等操作的最佳位置。其默认实现是终止所有的子actor并调用 :meth:`postStop` 。
2. 最初 ``actorOf`` 调用时传入的工厂方法被用来创建新的实例。
3. 新actor的 :meth:`postRestart` 方法被调用，参数中包含导致重启的异常信息。默认情况下会调用 :meth:`preStart`，跟正常启动的情况相同。

actor的重启仅会替换掉实际的actor对象; 信箱的内容不会被重启所影响, 所以对消息的处理将在 :meth:`postRestart` 钩子返回后继续进行. 触发异常的消息不会被重新接收。在actor重启过程中所有发送到该actor的消息将照例在信箱中排队。

.. warning::
    请注意，故障通知相对于用户消息的顺序是不确定的。特别地，父actor可能在处理子actor故障前发往父actor的最后一个消息之前将其重启。参见 :ref:`message-ordering` 了解详情。


停止钩子
---------

一个Actor停止后，它的 :meth:`postStop` 钩子将被调用，它可以用来完成诸如取消该actor在其它服务中的注册等工作。 这个钩子保证在该actor的消息队列被禁用后才运行， 也就是，发送到已停止的actor的消息会被重定向到 :obj:`ActorSystem` 的 :obj:`deadLetters` 中。


.. _actorSelection-java:

通过Actor Selection标识Actor
======================================

如 :ref:`addressing` 中所述, 每个actor都拥有一个唯一的逻辑路径, 此路径可通过从子actor到父actor跟踪actor链接，直到抵达actor系统的根为止，来获得。actor还拥有一个物理路径，如果监管链包含任何远程监管者，此路径可能会与逻辑路径不同。这些路径被系统用来查找actor，例如，当收到一个远程消息时，会查找收件者， 但是它们具有更直接的用处：actor可以通过指定绝对或相对路径-逻辑的或物理的-来查找其它的actor，并随结果收到一个 :class:`ActorSelection` :

.. includecode:: code/docs/actor/UntypedActorDocTest.java#selection-local

其中指定的路径被解析为一个 :class:`java.net.URI` , 它以 ``/`` 分隔成路径段. 如果路径以 ``/`` 开始则表示绝对路径，从根守卫actor ( ``"/user"`` 的父actor)开始查找; 否则从当前actor开始。如果某一个路径段为 ``..`` , 会向“上”一级到达其监管者，否则将向“下”一级找到对应名称的子actor。 必须注意的是 actor路径中的 ``..``  总是表示逻辑结构，也就是其监管者。 

actor选择的路径元素可能会包含通配符，允许向那一段actor广播消息。

.. includecode:: code/docs/actor/UntypedActorDocTest.java#selection-wildcard

消息可以通过 :class:`ActorSelection` 来发送，并且 :class:`ActorSelection` 的路径会在发送每条消息时被查找. 如果这个selection不匹配任何actor，则消息会被丢弃。

要获得一个 :class:`ActorSelection` 所对应的 :class:`ActorRef`，你需要向这个selection发送一条消息，并且使用回复消息中的 ``getSender`` 引用。 有一个内置的 ``Identify`` 消息，所有Actor都能理解，并且自动地回复一个 ``ActorIdentity`` 消息，包含 :class:`ActorRef` 。这条消息由所经过的actor进行特殊处理，如果一次具体的按名称查找失败（也就是一个非通配符的路径元素并不对应于一个活着的actor），那个会生成一个否定响应。请注意这并不意味着该响应的投递是有保证的，它仍是一条普通消息。

.. includecode:: code/docs/actor/UntypedActorDocTest.java#import-identify
.. includecode:: code/docs/actor/UntypedActorDocTest.java#identify

你还可以使用 :class:`ActorSelection` 的 ``resolveOne`` 方法获取一个 :class:`ActorSelection` 所对应的 :class:`ActorRef`。它会返回一个 ``Future`` ，包含所匹配的 :class:`ActorRef` ，如果这样的actor存在的话。如果这样的actor不存在，或者在所给的 `timeout` 之内没有完成，则会以一个错误 [[akka.actor.ActorNotFound]] 而告终。

如果开启了 :ref:`remoting <remoting-java>` ，则远程actor地址也可以被查找。: 

.. includecode:: code/docs/actor/UntypedActorDocTest.java#selection-remote

一个阐述远程actor查找的例子是 :ref:`remote-sample-java`.

.. note::
  ``actorFor`` 已被废弃，推荐使用 ``actorSelection`` ，因为通过 ``actorFor`` 所获取的引用，对于本地和远程actor具有不同的行为。
  在本地actor的情形下，所指的actor需要在查找之前存在，否则所获取的引用会是一个 :class:`EmptyLocalActorRef`。
  即使具有相同路径的actor在获取actor引用之后被创建，这也成立。
  对于使用 `actorFor` 获取的远程actor引用，其行为是不同的，向这样一个引用发送消息，在后台针对每次消息发送都会根据路径在远程系统中查找actor。

消息与不可变对象
=========================

**IMPORTANT**: 消息可以是任何类型的对象，但必须是不可变的。（目前） Akka还无法强制不可变，所以这一点必须作为约定。

以下是一个不可变消息的例子:

.. includecode:: code/docs/actor/ImmutableMessage.java#immutable-message

发送消息 Send messages
=============

向actor发送消息需要使用下列方法之一。

* ``tell`` means “fire-and-forget”, 例如，异步发送一个消息并立即返回。
* ``ask`` 同步发送一个消息，并且返回一个 :class:`Future` 表示可能的回复。

消息的顺序是基于每个发送者来单独保证的。

.. note::

	使用 ``ask`` 隐含了一些性能开销，因为必须对超时进行跟踪，且需要将一个 ``Promise`` 桥接为一个 ``ActorRef`` ，并且它必须在远程环境下可访问。 因此为了性能，应当总是优先使用，``tell`` ，仅在迫不得已的情况下才使用 ``ask`` 。

在所有这些方法中，你可以选择传递哪个 ``ActorRef`` 。这样做仅供练习，这将允许接收消息的actor能够回复你的消息，因为sender引用同消息一起发送。

.. _actors-tell-sender-java:

Tell: Fire-forget
-----------------

这是发送消息的推荐方式。 不会在等待消息时阻塞。它会带来最好的并发性和可伸缩性。

.. includecode:: code/docs/actor/UntypedActorDocTest.java#tell

发送者引用会随着消息而传递，并且在接收者actor处理消息时可通过 :meth:`getSender()` 方法访问。 再actor内部，发送者总应当是 :meth:`getSelf` , 但有些情况下回复消息应当被路由到其他actor—例如父actor—在这种情况下 :meth:`tell` 的第二个参数应当会有所不同。 在actor外部，并且不需要回复，则第二个参数可以为 ``null``; 如果在actor外部发送消息时需要回复，那么你可以使用下面讲到的ask模式。 

Ask: 发送和接收Future
----------------------------

``ask`` 模式既涉及actor又涉及future, 所以它是作为一种使用模式而提供，而不是  :class:`ActorRef` 的一个方法:

.. includecode:: code/docs/actor/UntypedActorDocTest.java#import-ask
.. includecode:: code/docs/actor/UntypedActorDocTest.java#ask-pipe

上面的例子展示了将 ``ask`` 模式同future上的 ``pipeTo`` 模式一起展示，因为这是一种常用的组合。 请注意上面所有的调用都是完全非阻塞和异步的： ``ask`` 产生 :class:`Future`, 使用 :meth:`Futures.sequence` 和 :meth:`map` 方法将两个future组合成一个新的future，然后 ``pipe`` 安装了一个 ``onComplete`` -处理函数到这个future， 导致合并后的  :class:`Result` 被提交到另一个actor。

使用 ``ask`` 将会跟 ``tell`` 一样发送消息给接收方actor, 并且接收方必须通过 ``getSender().tell(reply, getSelf())`` 来发送回复，从而使用一个值来完成所返回的 :class:`Future` 。 ``ask`` 操作涉及创建一个内部actor来处理回复，这需要设置一个超时时间，在此之后销毁这个actor，从而避免资源泄漏；在下面查看更多信息。

.. warning::

    如果要以异常来结束一个future，你需要发送一个 Failure 消息到发送方。当actor处理消息时抛出异常，这个操作 *不会自动完成* 。

.. includecode:: code/docs/actor/UntypedActorDocTest.java#reply-exception

如果一个actor没有完成future, 它会在作为 ``ask`` 参数传入的超时时间之后到期；这会以 :class:`AskTimeoutException` 来完成 :class:`Future` 。

参阅 :ref:`futures-java` 以了解更多关于等待和查询future的信息。

``Future`` 的 ``onComplete`` , ``onResult`` 或 ``onTimeout`` 方法用来注册一个回调，以便在Future完成时得到通知。这给了你一种避免阻塞的方式。 

.. warning::

   在使用future回调时, 在actor内部你要小心避免捕获该actor的引用, 也就是不要在回调中调用该actor的方法或访问其可变状态。这会破坏actor的封装，并可能引入同步bug和竞态条件， 因为回调与此actor会被并发地调度。不幸的是目前还没有一种编译时的方法能够探测到这种非法访问。 另外请参阅: :ref:`jmm-shared-state`。

转发消息
---------------

你可以将消息从一个actor转发给另一个。这意味着虽然经过了一个‘中介’，但最初的发送者地址/引用将保持不变。当实现用作路由器、负载均衡器、备份等的actor时，这会很有用。 你还需要传递你的context变量。

.. includecode:: code/docs/actor/UntypedActorDocTest.java#forward

接收消息
================

当Actor收到消息时，消息会被传入 ``onReceive`` 方法, 这是 ``UntypedActor`` 基类的一个抽象方法，在Actor中必须被定义。

这是一个例子：

.. includecode:: code/docs/actor/MyUntypedActor.java#my-untyped-actor

if-instanceof 检查的一个替代方案是使用 `Apache Commons MethodUtils
<http://commons.apache.org/beanutils/api/org/apache/commons/beanutils/MethodUtils.html#invokeMethod(java.lang.Object,%20java.lang.String,%20java.lang.Object)>`_ 来调用一个参数类型与消息类型匹配的指定名称的方法。

.. _UntypedActor.Reply:

回复消息
=================

如果你需要一个用来回复消息的句柄，可以使用 ``getSender()`` , 它是一个ActorRef。 你可以用 ``getSender().tell(replyMsg, getSelf())`` 向这个引用发送消息来进行回复. 你也可以将这个Actor引用保存起来，用于以后回复。如果没有sender (消息不是使用actor或future上下文来发送的) 那么 sender默认为 ‘dead-letter’ actor引用。

.. includecode:: code/docs/actor/UntypedActorDocTest.java#reply
   :exclude: calculate-result

消息接收超时
===============

`UntypedActorContext` 的 :meth:`setReceiveTimeout` 方法可定义一段不活跃超时时间，在此之后会触发一个 `ReceiveTimeout` 消息发送。如果指定的话，接收函数应当能够处理 `akka.actor.ReceiveTimeout` 消息。
1 毫秒是所支持的最短超时时间。

请注意接收超时可能刚好在另一个消息入队之后引起 `ReceiveTimeout` 消息并将其入队；因此**并不能保证**接收超时消息之前的空闲时间一定跟方法中所配置的超时时间吻合。

一旦设置之后，接收超时会保持有效（也就是在不活跃时间之后重复发生）。传入 `Duration.Undefined` 可以关闭这项功能。

.. includecode:: code/docs/actor/MyReceiveTimeoutUntypedActor.java#receive-timeout

.. _stopping-actors-java:

停止Actor
===============

actor的终止是通过调用``ActorRefFactory``，即 ``ActorContext`` 或 ``ActorSystem`` 的 :meth:`stop` 方法。 通常context被用来停止子actor，而system用来停止顶层actor. actor实际的停止操作是异步执行的， 也就是 :meth:`stop` 可能在actor被停止之前返回。

如果当前有正在处理的消息，对该消息的处理将在actor停止之前完成，但是信箱中另外的消息将不会被处理。默认情况下这些消息会被送到 :obj:`ActorSystem` 的 :obj:`deadLetters` , 但是这取决于信箱的实现。 

actor的停止分两步: 第一步actor将暂停对信箱的处理，向所有子actor发送终止命令，然后处理来自子actor的终止通知，直到所有一个子actor已停止， 最后终止自己 (调用 :meth:`postStop` , 清空信箱, 向 :ref:`DeathWatch <deathwatch-java>` 发布 :class:`Terminated` , 告知其监管者)。 这个过程保证actor系统中的子树以一种有序的方式终止, 将终止命令传播到叶子结点并将其确认消息收集到被停止的监管者。如果其中某个actor没有响应 (也就是，由于处理一条消息用了太长时间以至于没有收到停止命令), 则整个过程将会被卡住。

在 :meth:`ActorSystem.shutdown()` 被调用时, 系统根监管actor会被停止，以上所述的过程将保证整个系统的正确终止。 

:meth:`postStop()` 钩子是在actor被完全停止以后调用的。这允许清理资源:

.. includecode:: code/docs/actor/UntypedActorDocTest.java#postStop
   :exclude: clean-up-resources-here

.. note::

    由于actor的终止是异步的, 你不能马上重用你刚刚终止的子actor的名字；这会导致 :class:`InvalidActorNameException` 。 你应该 :meth:`watch()` 正在终止的actor，并且在响应最终到达的 :class:`Terminated` 消息时创建它的替代者。

.. _poison-pill-java:

PoisonPill
----------

你也可以向actor发送 akka.actor.PoisonPill 消息, 这个消息处理完成后actor会被终止。 PoisonPill跟普通消息一样被放进队列，因此会在已经进入信箱队列的其它消息之后被处理。

用法如下：

.. includecode:: code/docs/actor/UntypedActorDocTest.java
   :include: poison-pill

优雅地终止 Graceful Stop
-------------

如果你想等待终止过程的结束，或者编排若干actor的有序终止，可以使用 
:meth:`gracefulStop` :

.. includecode:: code/docs/actor/UntypedActorDocTest.java
   :include: import-gracefulStop

.. includecode:: code/docs/actor/UntypedActorDocTest.java
   :include: gracefulStop

.. includecode:: code/docs/actor/UntypedActorDocTest.java
   :include: gracefulStop-actor

当 ``gracefulStop()`` 成功返回时, actor的 ``postStop()`` 钩子将已执行: 在 ``postStop()`` 末尾和 ``gracefulStop()`` 的返回之间存在一个 happens-before 边界。

在上例中，一个自定义的 ``Manager.SHUTDOWN`` 被发送给目标actor来开启停止actor的过程。你可以使用 ``PoisonPill`` 来做到这一点，但那样你在停止目标actor之前就只能同其他actor进行可能性很有限的交互。简单的清理任务可以在 ``postStop`` 中处理。.

.. warning::

  记住一点，actor的停止和它的名称被注销，是独立的事件，相互异步地发生。因此你可能发现在 ``gracefulStop()`` 返回之后，名称仍然被占用。为了保证正确的注销，应当只从你能够控制的监管者内重用名称，并且只在 :class:`Terminated` 消息的响应中进行，也就是，不适用于顶层actor。

.. _UntypedActor.HotSwap:

HotSwap
=======

Upgrade
-------

Akka支持在运行时热插拔Actor的消息循环 (例如其实现)。从Actor内部使用 ``getContext().become`` 方法。 热插拔的代码保留在一个堆栈中，可以进行push (替换栈顶或添加到栈顶) 和 pop操作.

.. warning::
  请注意actor被监管者重启时会恢复到起始的行为。

使用 ``getContext().become`` 来热插拔Actor:

.. includecode:: code/docs/actor/UntypedActorDocTest.java
   :include: import-procedure

.. includecode:: code/docs/actor/UntypedActorDocTest.java
   :include: hot-swap-actor

:meth:`become` 方法的变化形式用于多种不同的东西， 例如实现一个有限状态自动机(FSM). 它会替换当前行为 (也就是，行为堆栈的栈顶), 这意味着你不需要使用:meth:`unbecome`, 相反地总是将下一个行为显式地安装。

另一种使用 :meth:`become` 的方式并不是替换，而是向栈顶添加元素。在这种情况下，必须仔细确保 “pop”操作 (也就是， :meth:`unbecome` ) 的数目最终匹配“push” 操作的数目，否则这实际上是内存泄漏 (因此这种行为不是默认的).

.. includecode:: code/docs/actor/UntypedActorSwapper.java#swapper

Stash
=====

``UntypedActorWithStash`` 类允许actor临时隐藏那些actor当前的行为不能或不应当处理的消息。 当更改actor的消息处理器时，也就是刚好在调用 ``getContext().become()`` 或 ``getContext().unbecome()`` 之前，所有隐藏的消息可以变为"非隐藏"（unstashed）, 于是将其插入到actor信箱的最前端。这样，隐藏的消息将按照最初接收的顺序来进行处理，一个继承 ``UntypedActorWithStash`` 的actor会自动获得一个基于双端队列的信箱。

.. note::

    抽象类 ``UntypedActorWithStash`` 实现了标记接口 ``RequiresMessageQueue<DequeBasedMessageQueueSemantics>`` ，请求系统为actor自动选择一个基于双端队列的信箱实现，参见信箱文档: :ref:`mailboxes-java`.

这里是 ``UntypedActorWithStash`` 类的一个实战示例:

.. includecode:: code/docs/actor/UntypedActorDocTest.java#import-stash
.. includecode:: code/docs/actor/UntypedActorDocTest.java#stash

调用 ``stash()`` 添加当前消息 (actor最新收到的消息) 到actor的stash中。它通常在actor消息处理器的default分支进行调用，从而将其他分支未处理的消息隐藏起来。将同一条消息隐藏两次是非法的；这么做会导致一个 ``IllegalStateException`` 被抛出。stash还可以是有界的，在这种情况下调用 ``stash()`` 可能导致超出容量, 这将导致一个 ``StashOverflowException`` 。 stash 的容量可以通过信箱配置中的 ``stash-capacity`` 设置项(一个 ``Int``)来进行配置。

调用 ``unstashAll()`` 会将stash中的消息放入actor的信箱队列，直到信箱的容量（如果设置了的话）耗尽（注意来自stash中的消息会放到信箱头部）。 万一信箱溢出，一个 ``MessageQueueAppendFailedException`` 会被抛出。调用 ``unstashAll()`` 之后，stash保证为空。

stash是由一个 ``scala.collection.immutable.Vector`` 来支撑的。 因此，可隐藏特别大量的消息，而不会明显影响性能。

注意stash是actor临时状态的一部分，不同于信箱。因此，它应该像actor状态中的其它具有相同性质的部分一样管理。 :class:`UntypedActorWithStash` 所实现的 :meth:`preRestart` 会调用 ``unstashAll()`` ，这通常是符合所需的行为。

.. note::

  如果你想强制你的actor只使用无边界的stash，那么你应当转而使用 ``UntypedActorWithUnboundedStash`` 。


.. _killing-actors-java:

杀死actor Killing an Actor
================

你可以发送 ``Kill``  消息来杀死actor， 这将导致actor抛出一个 :class:`ActorKilledException`, 触发一个错误。 这个actor会暂停其操作，并且其监管者会被询问如何处理这个错误， 这可能意味着继续这个actor，重启它，或者完全终止它。参见 :ref:`supervision-directives` 了解更多信息。

像如下这样使用 ``Kill`` ：

.. includecode:: code/docs/actor/UntypedActorDocTest.java
   :include: kill

Actor 与 异常
=====================

在消息被actor处理的过程中可能会抛出异常，例如数据库异常。 

消息会受到什么影响
---------------------------

如果消息处理过程中（即从信箱中取出并交给当前行为后）发生了异常，这个消息将丢失。必须明白它不会被放回到信箱中。所以如果你希望重试对一个消息的处理，你需要自己捕获异常，并且重试你的代码。请确保限制了重试的次数，因为你不会希望系统出现活锁 (从而消耗大量CPU循环，却毫无进展)。 另一种可能是查看 :ref:`PeekMailbox pattern <mailbox-acking>`.

信箱会受到什么影响
---------------------------

如果消息处理过程中发生异常，信箱不会发生任何变化。如果actor被重启，将使用相同的信箱。因此信箱中的所有消息都还在。 

actor会受到什么影响
-------------------------

如果actor中的代码抛出了异常，此actor会被暂停，并且监管流程会启动（参见 :ref:`supervision` ）取决于监管者的决定，这个actor会被继续（就像什么都没有发生），重启（擦除其内部状态并从头开始）或终止。

初始化模式
=======================

Acotr所提供的丰富的生命周期钩子方法提供了实现各种各样的初始化模式的实用工具箱。在 ``ActorRef`` 的一生中, 一个actor可能会经历多次重启，其中旧的实例被一个全新的实例取代，对于只能看到 ``ActorRef`` 的外界，重启是不可见的。

可以认为新实例是一个 "化身". 初始化可能对于actor的每个化身都是必须的，但有时会需要让初始化只在 ``ActorRef`` 创建时产生的第一个实例上进行。接下来的小节提供了针对不同初始化需求的模式。

通过构造器进行初始化
------------------------------

使用构造器进行初始化有很多好处。首先，它使得 ``val`` 字段能够被用来保存acotr实例一生中不会改变的任何状态，使得actor的实现更加假装。构造器对actor的每个化身都会调用，因此actor内部总是可以假设正确的初始化已经进行。这也是此方式的缺点，因为有时可能想避免再重启时重新初始化内部状态。例如，在重启时保留子actor常常是有用的。下一节针对这种情况提供了一种模式。

通过preStart完成初始化
---------------------------

actor的 ``preStart()`` 方法只在第一个实例初始化时被直接调用一次，也就是，在创建其 ``ActorRef`` 时。 在重启的情况下, ``preStart()`` 是在 ``postRestart()`` 中调用的， 因此如果不覆盖的话， ``preStart()`` 会在每个化身上被调用。然而，覆盖 ``postRestart()`` 能够关闭这种行为，并确保 ``preStart()`` 仅被调用一次。

这种模式的一种用途是禁止在重启时为子actor创建新的 ``ActorRef``。 这可通过覆盖 ``preRestart()`` 来实现:

.. includecode:: code/docs/actor/InitializationDocSpecJava.java#preStartInit

请注意，子actor *仍然会被重启* ，但是不会创建新的 ``ActorRef`` 。可以将相同的原则递归应用到子actor，确保其 ``preStart()`` 仅在创建其引用时会被调用一次。

更多信息请参见 :ref:`supervision-restart`.

通过消息传递进行初始化
----------------------------------

有些情况下，在构造方法中出啊如actor初始化所需的所有信息室不可能的，例如当存在循环依赖时。在这种情况下，actor应当监听初始化消息，并且使用 ``become()`` 或有限状态自动机的状态转换，来编码actor已初始化和未初始化的状态。

.. includecode:: code/docs/actor/InitializationDocSpecJava.java#messageInit

如果这个actor可能在其初始化之前接收消息，一个使用的工具是 ``Stash`` ，可将消息保存起来，直到初始化完成，并且在actor已初始化之后回放这些消息。

.. warning::

  这个模式应当谨慎使用，并且仅当上述模式都不适用时才采用。 一个可能存在的问题是，消息被发往远程actor时可能会丢失。而且，在未初始化的状态下发布一个 ``ActorRef`` 可能导致它在初始化完成之前接收到用户消息。
