.. _howto-java:

######################
HowTo: 常见的模式
######################

本节列出常见的actor模式，它们被发现是有用的，优雅的，或者有教育意义的。任何东西都是受欢迎的，范例话题是消息路由策略，监控模式，重启处理，等等。作为一个特殊的奖励，对本节的添加将标上贡献者的名字，并且如果每个Akka用户在其代码中发现一个重复出现的模式时都能为了大家受益而分享出来，将是很好的事情。如果恰当的话，将其添加到 ``akka.pattern`` 包，用于创建一个 `OTP-like library
<http://www.erlang.org/doc/man_index.html>`_ 也是合适的。

你可能发现Scala中的
:ref:`howto-scala` 一章所描述的某些模式是有用的，尽管这些示例代码是用Scala编写的。

调度周期性的消息
============================

这个模式描述了如何按照两种不同的方式将周期性的消息调度给自己。

第一种方式是在actor的构造器中建立周期性消息调度，并且在 ``postStop`` 中取消这一调度，否则我们可能有多个注册的消息被发送到同一个actor。

.. note::
   
   使用这种方式，被调度的周期性消息的发送，将在actor重启时重新开始。这也意味着，重启过程中，两个tick消息之间所流逝的时间可能基于你重启被调度的消息发送相对于最后一个消息被发送的时间，以及初始延时的大小，而发生漂移。最坏的情况下，这是 ``interval`` 加 ``initialDelay``.

.. includecode:: code/docs/pattern/SchedulerPatternTest.java#schedule-constructor

第二种变化形式是在actor的 ``preStart`` 方法中建立一个初始的一次性消息发送，然后这个actor在收到这条消息时建立一个新的一次性消息发送。你还可以覆盖 ``postRestart`` ，这样我们不需要再次调用 ``preStart`` 并且再次调度初始的消息发送。

.. note::
   使用这种方式，我们不会在actor压力很大时用tick消息填满信箱，而仅当我们看到前一个tick消息时调度一个新的tick消息。

.. includecode:: code/docs/pattern/SchedulerPatternTest.java#schedule-receive

具有高级错误报告机制的一次性Actor树
======================================================

*Contributed by: Rick Latrine*

从Java进入actor世界的一个好方法是使用 Patterns.ask() 。这个方法会启动一个临时的actor来转发这条消息，并收集来自被 "ask" 的actor的回复。当被ask的actor内部发生错误时，默认的监管处理将会接管。Patterns.ask() 的地爱哦用这 *不* 会被通知。

如果调用者对这样的异常感兴趣，它必须确保被ask的actor用Status.Failure(Throwable) 来进行回复。为了完成异步的工作，在被ask的actor背后可能催生了一个复杂的actor层次结构。而监管是控制错误处理的已有方法。

可惜被ask的actor必须了解监管，并且必须捕获异常。这样的actor不太可能在一个不同的actor层次结构中被重用，并且包含弱化的 try/catch 代码块。

这个模式提供一种封装临时actor的监管和错误传播的方式。最终Patterns.ask()返回的promise被实现为一个失败，包含异常。

让我们看看示例代码：

.. includecode:: code/docs/pattern/SupervisedAsk.java

在这个 askOf 方法中，用户消息被发送到 SupervisorCreator 。SupervisorCreator 创建了一个 SupervisorActor 并且转发这条消息。 这避免了actor系统由于actor创建而超负荷。SupervisorActor 负责创建用户actor，转发消息，处理actor终止和监管。另外如果执行时间耗尽，SupervisorActor 会停止用户actor。

在发生异常的情况下，监管者告诉临时actor所抛出的是哪个异常。之后actor层次结构会被停止。

最终我们能够执行一个actor并且接收结果或异常。

.. includecode:: code/docs/pattern/SupervisedAskSpec.java

模板模式
================

*Contributed by: N. N.*

这个是特别号的模式，因为它甚至带有一些空白的示例代码：

.. includecode:: code/docs/pattern/JavaTemplate.java
   :include: all-of-it
   :exclude: uninteresting-stuff

.. note::
   
   传播这句话： 这是最简单的出名方式！

请将这个模式保留在文件末尾。
