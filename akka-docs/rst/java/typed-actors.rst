有类型Actors
===================

Akka 有类型Actor 是对 `Active Objects <http://en.wikipedia.org/wiki/Active_object>`_ 模式的一种实现。实质上会将方法调用变为异步分发，而不是Smalltalk出现时默认的同步分发。

有类型Actor 包含2个 "部件", 一个公开接口，以及一个实现，而且如果你曾经使用 "企业级" Java做过事情， 这对你来说应当非常熟悉。 如同正常的Actor，有一个外部API (公共接口实例) ，会将方法调用异步地委托给实现类的私有实例。

有类型Actors 与 （无类型）Actor 相比，优势在于使用 TypedActor 会让你拥有静态契约，并且不需要定义自己的消息，劣势在于它对你能做什么不能做什么施加了一些限制，也就是你不能使用 become/unbecome.

有类型Actor的实现用到了 `JDK Proxies <http://docs.oracle.com/javase/6/docs/api/java/lang/reflect/Proxy.html>`_ ，它提供了一个拦截方法调用的非常简单易行的API。

.. note::

    如同正规的Akka Untyped Actor, 有类型Actor一次处理一次调用。

何时使用有类型Actors
------------------------

有类型actor对于桥接acotr系统（“内部”）和非actor代码（“外部”）非常有用，因为它们允许你在外部编写通常的OO风格代码。 将它们看作门：它们实际上位于私有空间和公共场所的交界，但是你在自己的屋子里不想要很多门，难道想要吗？ 一个更长的讨论请参见 `this blog post <http://letitcrash.com/post/19074284309/when-to-use-typedactors>`_.

更多的背景知识：TypedActor很容易被误用为RPC, 而这个抽象被 `公认为
<http://doc.akka.io/docs/misc/smli_tr-94-29.pdf>`_ 有漏洞。因此当我们提到让高度可伸缩的并行软件变得更易于编写时，TypedActor 并不是我们首先考虑的。它们有它们的用场，保守地使用它们。

行业工具
----------------------

在我们开始创建第一个有类型Actor之前，我们首先应当回顾我们所掌握的工具，它位于 ``akka.actor.TypedActor``.

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-extension-tools

.. warning::
	跟不要暴露Akka Actor的、 ``this`` 类似, 不要暴露一个有类型Actor 的 ``this`` 也是很重要的。相反地，你可以传递外部代理引用，它可从你的有类型Actor中通过 ``TypedActor.self()`` 获取。这是你的外部身份，正如 ``ActorRef``是一个Akka Actor的外部身份。

创建有类型Actor
---------------------

要创建有类型Actor，你需要拥有一个或多个接口，以及一个实现。

假定有如下导入：

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: imports

一个示例接口：

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-iface
   :exclude: typed-actor-iface-methods

此接口的示例实现:

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-impl
   :exclude: typed-actor-impl-methods

创建我们的 ``Squarer`` 的一个有类型Actor实例的最琐碎方式为 :

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-create1

第一个类型是代理类，第二个类型是实现类。如果你需要调用一个指定的构造器，你需要这么做：

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-create2

由于你提供了一个 ``Props``, 你可以指定索要使用的dispatcher, 默认超时时间，等等。现在，我们的 ``Squarer`` 不包含任何方法，因此我们最好添加这些：

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-iface

好了，现在我们已经有了一些可以调用的方法，但我们需要在 ``SquarerImpl`` 中进行实现。

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-impl

非常好，现在我们拥有一个接口以及它的一个实现，并且我们知道如何用这些创建一个有类型Acto因此让我们看看如何调用这些方法。

方法分发语义
-------------------------

方法返回：

  * ``void`` 将按照 ``fire-and-forget`` 语义进行分发, 跟 ``ActorRef.tell`` 完全类似
  * ``scala.concurrent.Future<?>`` 将使用 ``send-request-reply`` 语义, 跟 ``ActorRef.ask`` 完全类似
  * ``akka.japi.Option<?>`` 将使用 ``send-request-reply`` 语义, 但 *会* 在等待答案时阻塞，而且会在超时之前没有产生答案时则返回 ``akka.japi.Option.None`` 否则 返回 ``akka.japi.Option.Some<?>`` ，其中包含计算结果。任何在此调用期间抛出的异常会被重新抛出。
  * 任何其他类型的返回值会使用 ``send-request-reply`` 语义, 但是 *会* 阻塞并等待答案，如果出现超时则抛出 ``java.util.concurrent.TimeoutException`` ，或者当调用时抛出异常，则将其重新抛出。

消息和不可变性
-------------------------

尽管 Akka 无法强制有类型Actor 的方法参数是不可变的，但是我们 *强烈* 推荐传入的参数为不可变对象。

单向消息发送
^^^^^^^^^^^^^^^^^^^^

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-call-oneway

如此简单! 这个方法会由另一个线程执行; 异步地.

请求-响应 消息发送
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-call-option

这将阻塞的时间会跟有类型Actor的 ``Props`` 中所设置的超时时间一样长，如果需要设置的话。 当超时发生时它会返回 ``None`` 

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-call-strict

这将阻塞的时间会跟有类型Actor的 ``Props`` 中所设置的超时时间一样长，如果需要设置的话。 当超时发生时它会抛出一个 ``java.util.concurrent.TimeoutException``。

使用future的请求-响应 消息发送
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-call-future

次调用是异步的，并且返回的Future可被用于异步组合。

停止有类型Actor
---------------------

既然 Akka的有类型Actor 由Akka Actor支撑，当它们不再被需要时需要被停止。

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-stop

这会尽快地异步停止指定代理对象所关联的有类型Actor。

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-poisonpill

这会异步停止指定代理对象所关联的有类型Actor， 在其处理完此调用之前所有的调用之后。

有类型Actor层次结构
-----------------------

由于你可以通过传入一个 ``ActorContext`` 来获取一个带上下文的有类型Actor扩展，你可以通过调用那个扩展上的 ``typedActorOf(..)`` 来创建一个有类型子Actor。

.. includecode:: code/docs/actor/TypedActorDocTest.java
   :include: typed-actor-hierarchy

你也可以在标准的Akka Actor中通过将 ``UntypedActorContext`` 作为TypedActor.get(…)的输入参数，来创建一个有类型子actor。

监管策略
-------------------

通过让你的有类型Actor实现类实现 ``TypedActor.Supervisor`` ，你可以定义监管子actor的策略，如 :ref:`supervision` 和 :ref:`fault-tolerance-java` 所述。

接收任意消息
--------------------------

如果你的TypedActor的实现类继承 ``akka.actor.TypedActor.Receiver`` ，则所有不是 ``MethodCall`` 的消息将被传入 ``onReceive``-方法.

这允许你对 DeathWatch 的 ``Terminated`` 消息以及其他类型的消息做出反应，例如，当与无类型actor交互时。

生命周期回调方法
-------------------

通过让你的有类型Actor实现类实现任何或全部的以下方法:

    * ``TypedActor.PreStart``
    * ``TypedActor.PostStop``
    * ``TypedActor.PreRestart``
    * ``TypedActor.PostRestart``

你可以介入你的有类型Actor的生命周期.

代理
--------

你可以使用 ``typedActorOf`` ，它需要一个 TypedProps 以及一个 ActorRef ，将给定的ActorRef代理为一个TypedActor。 这在你想要与其他机器上的TypedActor通信时，是非常有用的，只需将 ``ActorRef`` 传入到 ``typedActorOf`` 。

查找 & 远程处理
-----------------

由于 ``TypedActors`` 由 ``Akka Actors`` 来支撑， 你可以使用 ``typedActorOf`` 来代理可能位于其他节点上的 ``ActorRefs`` 。

.. includecode:: code/docs/actor/TypedActorDocTest.java#typed-actor-remote
