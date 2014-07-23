.. _migration-2.3:

################################
2.2.x 到 2.3.x 迁移指南
################################

2.3 版本包含一些结构变更，要求客户代码进行一些简单机械的源代码级别修改。

当从更早的版本迁移时，你首先应当遵循用于 :ref:`1.3.x 到 2.0.x <migration-2.0>` 以及 :ref:`2.0.x to 2.1.x <migration-2.1>`
以及 :ref:`2.1.x to 2.2.x <migration-2.2>` 的迁移说明。


取消了了集群单例中的数据转移
===========================================

在优雅离开的场景中支持从旧的单例向新的单例传递数据，这一功能已经被删除。有价值的状态应当被持久化到持久性存储中，例如使用 akka-persistence。 ``ClusterSingletonManager`` 的构造器/属性参数已经被修改为单例actor的普通 ``Props`` 参数，而不是工厂参数。

修改了集群自动关闭配置
=======================================

``akka.cluster.auto-down`` 设置项已经被替换为 ``akka.cluster.auto-down-unreachable-after``, 它让集群将不可达的节点自动标记为DOWN，在所配置的不可达时间之后。这一特性默认被禁用，跟2.2.x版本中一样。

在废弃阶段， ``akka.cluster.auto-down=on`` 被解释为立即自动关闭。

路由器
=======

路由器已被清理并加强。路由逻辑已被提取，从而在常规actor中也可以使用。一些易用性问题已经解决，例如正确拒绝无效的配置组合。Routee可以通过向路由器发送特殊管理消息来动态地增加和删除。

两类路由器被命名为 ``Pool`` 何 ``Group`` ，使之更加易于区分，并且减少对于其细微差异的混淆:

* Pool - 这种路由器将routee创建为子actor，并且在它们终止时将它们移除。
  
* Group - routee在路由器外部创建，并且路由器使用actor selection向指定路径发送消息，不监控其终止。

路由器的配置与2.2.x兼容，但是 ``router`` 类型更好的指定方式是，使用 ``-pool`` 或 ``-group`` 后缀.

某些用于编程定义路由器的类已经被改名，但是旧的类以废弃状态被保留。编译器将使用废弃警告来引导你例如 ``RoundRobinRouter`` 已被重命名为 ``RoundRobinPool`` 或 ``RoundRobinGroup`` ，取决于你实际使用哪种类型。


``SmallestMailboxRouter`` 没有组合routee路径的替代品，也就是group, 因为那一组合没啥用处。

一种可选的时代码更加可读的API加强，是使用 ``props`` 方法代替 ``withRouter`` 。 ``withRouter`` 没有废弃，你可以继续使用它，如果你更喜欢用那种方式来定义一个路由器。

Scala示例 ::

    context.actorOf(FromConfig.props(Props[Worker]), "router1")
    context.actorOf(RoundRobinPool(5).props(Props[Worker]), "router2") 

Java示例 ::

    getContext().actorOf(FromConfig.getInstance().props(Props.create(Worker.class)), 
      "router1");
      
    getContext().actorOf(new RoundRobinPool(5).props(Props.create(Worker.class)), 
        "router2");

为了让一个感知集群的路由器支持向多个routee路径发送消息，部署配置属性 ``cluster.routees-path`` 已被修改为字符串列表 ``routees.paths`` 属性。
旧的 ``cluster.routees-path`` 已废弃，但是在废弃阶段仍然可用。

示例 ::

    /router4 {
      router = round-robin-group
      nr-of-instances = 10
      routees.paths = ["/user/myserviceA", "/user/myserviceB"]
      cluster.enabled = on
    }

用于创建自定义路由器和缩放器的API已经变更，但是没有将旧的API以废弃的状态保留。这应当是仅被少数用户使用的API，并且他们迁移到新的API应当没有太多麻烦。

关于新router的知识参见 :ref:`documentation for Scala <routing-scala>` 和 
:ref:`documentation for Java <routing-java>` 。

Akka IO 不再为实验性的
=================================

Akka 2.2 引入的核心IO层现在是Akka完全支持的模块。

实验性的管道IO抽象已经被移除
======================================================

2.2中所引入的管道形式被发现没有太多指导意义，因此被中止。一些更加灵活易用的抽象将在未来取代它们的角色。管道在2.2系列中将仍然可用。

集群的expected-response-after 配置变化
=====================================================

配置属性 ``akka.cluster.failure-detector.heartbeat-request.expected-response-after``  已被重命名为 ``akka.cluster.failure-detector.expected-response-after``.

远程处理中的自动重试特性被删除，推荐使用retry-gate
====================================================================

retry-gate 特性现在是Remoting中唯一的错误处理策略。这一改变意味着，当远程检测到错误的连接时，会进入一个gated状态，其中所有缓存的以及后续的远程消息都会被丢弃，直到失败之后经过由配置键 ``akka.remote.retry-gate-closed-for`` 所配置的时间。这种行为放置了网络不稳定时的重连风暴和无限的缓存增长。在所配置的时间过去之后，门会被抬起，当存在需要发送的新的远程消息时，会尝试一次新的连接。

与这一变化相呼应，所有与旧的重连行为相关的配置(``akka.remote.retry-window`` 和
``akka.remote.maximum-retries-in-window``) 已被移除。

超时设置 ``akka.remote.gate-invalid-addresses-for`` ，用于控制特定失败事件的门时限，也被删除，所有门时限现在由 ``akka.remote.retry-gate-closed-for`` 配置项代为控制。

远程处理中为传输错误检测器减少了默认敏感性设置。
===============================================================================

由于最常用的远程传输方式是TCP，它提供了正确的连接终止时间，因此错误检测器敏感度设置 ``akka.remote.transport-failure-detector.acceptable-heartbeat-pause`` 现在的默认值是20秒，从而减少了负载所诱发的远程处理中的假阳性错误检测事件。为了应对使用一种非面向连接的协议的情况，推荐将这个选项以及 ``akka.remote.transport-failure-detector.heartbeat-interval`` 选项设置为一个更加敏感的值。 

Quarantine 现在是永久的
===========================

控制quarantine长度的配置项 ``akka.remote.quarantine-systems-for`` 已被删除。现在唯一可用的配置是 ``akka.remote.prune-quarantine-marker-after`` ，它影响了quarantine tombstones
保留的时间，从而避免长期内存泄漏。这个新选项的默认值为5天。

远程调用默认使用一个专有的dispatcher
===============================================

``akka.remote.use-dispatcher`` 的默认值已被修改为一个专有的dispatcher。

Dataflow已经废弃
======================

Akka 数据流已经被 `Scala Async <https://github.com/scala/async>`_ 取代。

持久化信箱已废弃
================================

持久化信箱已经由 ``akka-persistence`` 取代, 它提供了一些工具来支持可靠消息处理

更多关于 ``akka-persistence`` 的知识请参阅 :ref:`documentation for Scala <persistence-scala>` 和 :ref:`documentation for Java <persistence-java>`.

废弃的Agent STM 支持
=================================

Agent 参与外围STM事务是一个废弃特性。

Transactor 模块已经废弃
===============================

模块 ``akka-transactor`` 中actor和STM之间的集成已经废弃，并且将在未来版本中移除。

Typed Channel 已被移除
===============================

Typed channel 是一个试验性的特性，我们决定将其移除:它的实现依赖于Scala的一个试验性特性，在Java和其他语言中中没有对应物，并且其使用方式并不直观

被删除的废弃特性
===========================

下列之前已废弃的特性被删除:

 * `event-handlers renamed to loggers <http://doc.akka.io/docs/akka/2.2.3/project/migration-guide-2.1.x-2.2.x.html#event-handlers_renamed_to_loggers>`_ 
 * `API changes to FSM and TestFSMRef <http://doc.akka.io/docs/akka/2.2.3/project/migration-guide-2.1.x-2.2.x.html#API_changes_to_FSM_and_TestFSMRef>`_
 * DefaultScheduler 被 LightArrayRevolverScheduler 取代
 * 所有之前废弃的Props构造和析构方法
 
publishCurrentClusterState 已废弃
========================================

转而使用 ``sendCurrentClusterState`` 。 注意你还可以通过新的 ``Cluster(system).state`` 获取当前集群状态.


CurrentClusterState 不是一个 ClusterDomainEvent
===============================================

``CurrentClusterState`` 不再实现 ``ClusterDomainEvent`` 标记接口。

注意 ``Cluster.subscribe`` 新的 ``initialStateMode`` 参数，它允许将初始状态作为事件处理，而不是 ``CurrentClusterState``. 参见 
:ref:`documentation for Scala <cluster_subscriber_scala>` 和 
:ref:`documentation for Java <cluster_subscriber_java>`.


BalancingDispatcher 已废弃
=================================

使用 ``BalancingPool`` 代替 ``BalancingDispatcher``. 参见 :ref:`documentation for Scala <balancing-pool-scala>` 和 
:ref:`documentation for Java <balancing-pool-java>`.

在迁移期间你仍然可以通过在dispatcher配置中指定全类名来使用BalancingDispatcher ::

    type = "akka.dispatch.BalancingDispatcherConfigurator"

akka-sbt-plugin 被移除
==========================

用于打包应用程序二进制文件的 ``akka-sbt-plugin`` 已被删除。 版本 2.2.3 仍然可以被用来独立于应用程序的Akka版本而使用。版本 2.2.3 跟 sbt 0.12 和 0.13 都能一同使用。

`sbt-native-packager <https://github.com/sbt/sbt-native-packager>`_ 是使用sbt时创建Akka应用程序版本的推荐工具。

括号被添加到sender方法中
======================

括号被添加到Actor Scala API的 ``sender()`` 方法中，从而强调  ``sender()`` 引用不是引用透明的，并且不能被暴露到其他线程，例如使用future回调函数来捕获它。

推荐使用新的惯用法 ::

    sender() ! "reply"

然而，使用括号并不是强制的，你不需要修改任何代码。

ReliableProxy 构造器进行了修改
=================================

``akka-contrib`` 中的 ``ReliableProxy`` 构造器已被修改为接收一个 ``ActorPath`` 而不是 ``ActorRef``.  它还接收新的参数以支持重连。使用新的props工厂方法, ``ReliableProxy.props``.

Akka OSGi Aries Blueprint 已被删除
====================================

``akka-osgi-aries`` 已被删除. 如果需要的话可以在Akka之外实现。
 
TestKit: 重写了时间扩展
===============================

``TestDuration`` 已被修改到一个implicit value class 外加一个JavaTestKit中的 Java API 。请将 ::

    import akka.testkit.duration2TestDuration

修改为 ::

    import akka.testkit.TestDuration

