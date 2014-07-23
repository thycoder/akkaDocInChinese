.. _migration-2.4:

################################
2.3.x 到 2.4.x 迁移指南
################################

2.4 版本包含一些结构性变化，并且需要客户端代码进行一些简单机械的源代码级别修改。

当从较早版本迁移时，你首先应当遵循指令 :ref:`1.3.x to 2.0.x <migration-2.0>` ，然后是 :ref:`2.0.x to 2.1.x <migration-2.1>` ，然后是 :ref:`2.1.x to 2.2.x <migration-2.2>` ，再然后是 :ref:`2.2.x to 2.3.x <migration-2.3>`.

TestKit.remaining 抛出 AssertionError
=======================================

在Akka的较早版本中， `TestKit.remaining` 会返回默认的超时配置，位于 "akka.test.single-expect-default" 。这有点让人迷惑，因此它被修改为当在within之外调用时返回一个AssertionError。 然而旧的行为仍然可通过调用 `TestKit.remainingOrDefault` 来实现。

EventStream 和 ActorClassification EventBus 现在要求一个 ActorSystem
=======================================================================

``EventStream`` (:ref:`Scala <event-stream-scala>`, :ref:`Java <event-stream-java>`) 和 ``ActorClassification`` Event Bus (:ref:`Scala <actor-classification-scala>`, :ref:`Java <actor-classification-java>`) 现在都要求一个 ``ActorSystem`` 来正确运转。 原因是，远离有状态的内部生命周期符合用于退订已 ``Terminated`` 的完整响应式模型。

如果你实现了一个自定义的事件总线，你现在将需要通过构造器传入actor系统:

.. includecode:: ../scala/code/docs/event/EventBusDocSpec.scala#actor-bus

如果你曾手动创建EventStreams , 现在你必须提供一个actor系统，并且 *启动退订器*:

.. includecode:: ../../../akka-actor-tests/src/test/scala/akka/event/EventStreamSpec.scala#event-bus-start-unsubscriber-scala

请注意这一修改仅当你实现自己的总线的情况下会影响到你，Akka自身的 ``context.eventStream`` 仍然存在，并且不需要注意这一修改。

删除的废弃特性
===========================

下列之前废弃的特性已被删除:

* akka-dataflow

* akka-transactor

* 持久化信箱 (akka-mailboxes-common, akka-file-mailbox)

* Cluster.publishCurrentClusterState

* 在 Akka 2.3 中， akka.cluster.auto-down, 替换为 akka.cluster.auto-down-unreachable-after

* 旧的路由器及其配置。
  
  注意在路由器配置中你必须指定它是一个 ``pool`` 还是一个 ``group`` ，按照Akka 2.3中引入的方式。

* 不含时间单位的Timeout构造器
 
* JavaLoggingEventHandler, 替换为 JavaLogger

* UntypedActorFactory

* Java API TestKit.dilated, 迁移到 JavaTestKit.dilated

