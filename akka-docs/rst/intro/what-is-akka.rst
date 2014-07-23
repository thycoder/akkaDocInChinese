.. _what-is-akka:

###############
 Akka是什么?
###############

**可伸缩的实时事务处理**

我们相信编写出正确的并行，容错和可伸缩的应用程序是非常困难的。多数情况下，这是因为我们使用了错误的工具和错误的抽象级别。Akka就是用来改变那种状况的。使用Actor模型提升了抽象级别，为构建可伸缩，可容错(resilient)和响应式的应用程序提供了一个更好的平台——详情请参见 `Reactive
Manifesto <http://reactivemanifesto.org/>`_ 。在容错方面采取了“let it crash（任它崩溃）”模型，这种模型在电信行业的应用已经取得了巨大成功，构建了自我愈合的应用和永不停机的系统。Actor还为实现透明分布式提供了抽象机制，为编写真正的可伸缩可容错应用程序提供了基础。

Akka是开源的，可在遵循Apache2许可证的前提下获取。

下载网址为 http://akka.io/downloads.

请注意，所有的代码范例都是能够成功编译的，因此如果你想直接访问源码，请查看github上的Akka Docs 子项目:  `Java <@github@/akka-docs/rst/java/code/docs>`_ 
和 `Scala <@github@/akka-docs/rst/scala/code/docs>`_.


Akka实现了一组独特的混合物
===============================

Actors
------

Actor向你提供：

- 对并发和并行的简单和高级抽象。
- 异步，非阻塞，高性能的事件驱动编程模型。
- 非常轻量级的事件驱动进程(每GB堆内存可包含几百万个actor)

参见针对 :ref:`Scala <actors-scala>` 或 :ref:`Java <untyped-actors-java>` 的相应章节。

容错性
---------------

- 带有“let-it-crash”语义的监管层次结构。
- 监管层次结构可以跨越多个JVM，从而提供真正容错的系统。
- 非常适合于编写高度容错的自我愈合，永不停机的系统。

参见 :ref:`Fault Tolerance (Scala) <fault-tolerance-scala>` 和 :ref:`Fault Tolerance (Java) <fault-tolerance-java>`.

位置透明性
---------------------
Akka中所有的东西都被设计为可在分布式环境中工作：actor之间所有的交互都通过纯粹的消息传递来进行，并且一切都是异步的。

对于集群支持的概览，请参见 :ref:`Java <cluster_usage_java>`
和 :ref:`Scala <cluster_usage_scala>` 文档章节。

持久化
-----------

actor所接收的消息可选择进行持久化，并且在actor启动或重启时回放。这使得即使在JVM崩溃或actor迁移到另一个节点的情况下，actor也能恢复其状态。


更多详情参见相应章节 :ref:`Java <persistence-java>` 或 :ref:`Scala <persistence-scala>`.

Scala 和 Java APIs
===================

Akka 同时具有 :ref:`scala-api` 和 :ref:`java-api`.


Akka可以通过两种不同的方式使用
======================================

- 作为一个类库：由web应用使用，放到 ``WEB-INF/lib`` 目录中，或者作为你的classpath上的一个常用Jar包
- 作为一个微内核：独立的内核，你的应用程序可以放到其中。


参见 :ref:`deployment-scenarios` 了解更多详情。

商业支持
==================

Akka来自Typesafe Inc. 采用一个商业许可证，包含开发或产品支持，点击这里了解更多: `here
<http://www.typesafe.com/how/subscription>`_.

