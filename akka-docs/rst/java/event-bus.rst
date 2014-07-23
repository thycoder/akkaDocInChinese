.. _event-bus-java:

###########
 事件总线
###########

:class:`EventBus` 最初的构想是作为一种向actor分组发送消息的方式，现在已经被一般化为实现一个简单接口的一组抽象基类:

.. includecode:: code/docs/event/EventBusDocTest.java#event-bus-api

.. note::
    
	请注意EventBus不保存所发布的消息的发送者。如果你需要原始发送者的引用，你必须在消息内提供。

这种机制在Akka中用在不同的地方，例如 `事件流`_ 。实现代码可以利用下述的具体构件。

事件总线必须定义以下三个类型参数：

- :class:`Event` (E) 是所有发布到总线的事件的类型

- :class:`Subscriber` (S) 是允许注册到事件总线的订阅者的类型

- :class:`Classifier` (C) 定义里调度事件时选择订阅者的分类器

下面的trait在这些类型上仍然是泛型的，但是对于具体实现，它们必须被定义。

Classifier
===========

这里展示的分类器是Akka发布版本的一部分，但是如果不能找到完美匹配的一个，那么请自己实现，查阅已有分类器的实现的地方是 `github <@github@/akka-actor/src/main/scala/akka/event/EventBus.scala>`_ 

查找分类
---------------------

最简单的分类方式是，从每个事件中提取一个任意的分类器，并且为每个可能的分类器维护一个订阅者集合。这可被比作在无线电电台进行调谐。trait :class:`LookupClassification` 仍然是通用的，因为它的抽象层次在如何比较订阅者，以及具体如何分类之上。

必须实现的方法在下述例子中进行了阐述

.. includecode:: code/docs/event/EventBusDocTest.java#lookup-bus

这一实现的测试代码看起来类似这样：

.. includecode:: code/docs/event/EventBusDocTest.java#lookup-bus-test

这个分类方式是高效的，即使在一个特定事件没有订阅者的情况下。

子通道分类
-------------------------

如果分类器形成一个层次结构，并且需要订阅不仅可以在叶子节点进行，那么这种分类方式可能正好是恰当的。它可以被比作根据音乐流派来调频（可能多个）无线电频道。开发这种分类的目的是针对分类器仅是事件所对应的JVM类并且订阅者可能有兴趣订阅一个特定类的所有子类的情况，但是它可以同任何分类器层次结构一同使用。

下述例子中阐述了必须实现的方法:

.. includecode:: code/docs/event/EventBusDocTest.java#subchannel-bus

这一实现的一个测试类似如下:

.. includecode:: code/docs/event/EventBusDocTest.java#subchannel-bus-test

这个分类器在一个事件不能找到任何订阅者的情况下也是高效的，但是它使用常规的加锁机制来同步一个内部分类器缓存，因此它不适合于订阅关系频繁变化的使用场景（记住通过发送第一条消息来"开启"一个分类器也会重新检查所有之前的订阅）。

扫描分类
-----------------------

之前的分类器是针对具有严格层次结构的多分类器订阅而建构的，而这个分类器适用于存在重叠的，覆盖事件中各个部分，但没有形成层次结构的分类器的情形。它可以比作根据地理可达性来调频（可能多个）无线电台（用于古老的无线电传输）。


下述例子中阐述了必须实现的方法:

.. includecode:: code/docs/event/EventBusDocTest.java#scanning-bus

这一实现的一个测试类似如下:

.. includecode:: code/docs/event/EventBusDocTest.java#scanning-bus-test

这个分类器需要消耗的时间总是与订阅者数量成正比，与实际匹配数量无关。

.. _actor-classification-java:

Actor 分类
--------------------

这种分类方式最初被开发，是专门用于实现 :ref:`DeathWatch <deathwatch-java>`: 订阅者和分类器的类型都是 :class:`ActorRef`.

这种分类方式需要一个 :class:`ActorSystem` ，从而执行与订阅者相关的登记操作，作为Actor，订阅者可以在取消对EventBus的订阅之前终止。ActorClassification 维护一个系统Actor，它负责自动取消订阅终止的actor。

下述例子中阐述了必须实现的方法:

.. includecode:: code/docs/event/EventBusDocTest.java#actor-bus

这一实现的一个测试类似如下:

.. includecode:: code/docs/event/EventBusDocTest.java#actor-bus-test

这种分类器的事件类型仍然是泛型的，但是它对于所有使用场景都是高效的。

.. _event-stream-java:

事件流
============

事件流是每个actor系统的主要事件总线:它被用于承载 :ref:`log messages <logging-java>` 和 `死信`_ 并且可能被用于其他目的的用户代码所使用。它使用 `子通道分类`_ ，允许注册到相互关联的通道集合（正如 :class:`RemotingLifecycleEvent` 所使用的方式). 下述例子阐述了一个简单的订阅如何进行。给定一个简单的actor:

.. includecode:: code/docs/event/LoggingDocTest.java#imports-deadletter
.. includecode:: code/docs/event/LoggingDocTest.java#deadletter-actor

它可以如此被订阅:

.. includecode:: code/docs/event/LoggingDocTest.java#deadletters

类似于 `Actor 分类`_, :class:`EventStream` 会在终止时自动移除订阅者。

.. note::
   事件流是 *本地设施*, 意思是它在集群环境中 *不* 会将事件发布到其他节点（除非你显式地让一个远程Actor订阅这个流）。
   
   如果你需要在Akka集群中广播事件，*不* 能清楚了解你的接收者(也就是获取它们的ActorRef)，你可能想要查看: :ref:`distributed-pub-sub`.

默认处理器
----------------

actor系统启动时会创建并让actor订阅日志事件流：这里是 ``application.conf`` 中配置的处理器:

.. code-block:: text

  akka {
    loggers = ["akka.event.Logging$DefaultLogger"]
  }

这里列出的处理器是全限定类名，将订阅所有的日志事件，并且当运行时更改日志级别时，它们的订阅保持同步 ::

  system.eventStream.setLogLevel(Logging.DebugLevel());

这意味着一个级别的日志事件如果不会记录，就通常完全不会分配（除非对相应的事件类进行了手工订阅）

死信
------------

如 :ref:`stopping-actors-java` 中所述, 一个actor终止后进入队列或actor死后向其发送的消息会被重新路由到死信信箱，它默认会将这些消息包装在 :class:`DeadLetter` 中进行发布，这个被重定向的包装器保留信封的原始发送者，接收者和消息。 

其他用途
----------

事件流一直存在，并且能够使用，只需发布自己的时间（它接收 ``Object`` 类型），并且让一些监听器订阅想用的JVM类。

