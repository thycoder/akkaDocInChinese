.. _dispatchers-java:

Dispatcher
===================

Akka的 ``MessageDispatcher`` 让 Akka Actors "滴答", 可以说它是整个机器的引擎。所有 ``MessageDispatcher`` 实现同时也是一个 ``ExecutionContext``, 这意味着它们可以被用于执行任何代码，例如 :ref:`futures-java`.

默认dispatcher
------------------

每个 ``ActorSystem`` 都拥有一个默认的dispatcher，在没有为 ``Actor`` 配置其他dispatcher时会使用。 默认dispatcher可以配置，，默认情况下是带有指定的 ``default-executor`` 的一个 ``Dispatcher`` 。
如果一个ActorSystem创建时传入了一个ExecutionContext，则这个ExecutionContext将被用作ActorSystem中所有dispatcher的默认executor。 如果未指定 ExecutionContext, 则退而使用 ``akka.actor.default-dispatcher.default-executor.fallback`` 所指定的executor. 默认这是一个 "fork-join-executor", 它在大多数情况下都具有优异的性能。

.. _dispatcher-lookup-java:

查找一个Dispatcher
-----------------------

Dispatchers 实现了 :class:`ExecutionContext` 接口，因此可被用于运行 :class:`Future` 等.

.. includecode:: code/docs/dispatcher/DispatcherDocTest.java#lookup

为一个Actor设置dispatcher
-----------------------------------
因此假如你想为你的 ``Actor`` 配备默认之外的另一个, 你需要做两件事，其中第一件是配置这个:

.. includecode:: ../scala/code/docs/dispatcher/DispatcherDocSpec.scala#my-dispatcher-config

并且这里是另一个使用 "thread-pool-executor" 的例子:

.. includecode:: ../scala/code/docs/dispatcher/DispatcherDocSpec.scala#my-thread-pool-dispatcher-config
更多选项请查看 :ref:`configuration` 中default-dispatcher 一节。

然后你像平常那样创建一个actor，并且在部署配置中定义这个dispatcher。

.. includecode:: ../java/code/docs/dispatcher/DispatcherDocTest.java#defining-dispatcher-in-config

.. includecode:: ../scala/code/docs/dispatcher/DispatcherDocSpec.scala#dispatcher-deployment-config

部署配置的一种替代方案是在代码中定义dispatcher。如果你在部署配置中定义了 ``dispatcher`` ，那么这个值会被使用，而不是编程方式提供的参数。

.. includecode:: ../java/code/docs/dispatcher/DispatcherDocTest.java#defining-dispatcher-in-code

.. note::
    你在 ``withDispatcher`` 中所指定的dispatcher和部署配置中的 ``dispatcher`` 属性实际上是你的配置中的一个路径，因此在这个例子中它是个顶层小节，但是你还可以将其放到一个子小节中，其中你将使用句号（.）来表示子小节，像这样： ``"foo.bar.my-dispatcher"`` 
	
dispatcher 的类型
--------------------

消息dispatcher有4种不同类型

* Dispatcher

  - 这是一种基于事件的dispatcher，它将一组 Actors 绑定到一个线程池。如果没有没有指定，它是默认被使用的dispatcher。
  
  - 共享性质: 无限

  - 信箱: 任意类型, 每个Actor创建一个

  - 使用场景: 默认dispatcher, Bulkheading

  - 驱动: ``java.util.concurrent.ExecutorService``
               ，通过 "executor" 指定，可以使用 "fork-join-executor",
               "thread-pool-executor" 或 一个 ``akka.dispatcher.ExecutorServiceConfigurator`` 的完全类名。

* PinnedDispatcher

  - 这个dispatcher为使用它的每个actor分配一个唯一的线程; 也就是，每个actor将具有自己的线程池，并且其中只有一个线程。

  - 共享性质: None

  - 信箱: Any, creates one per Actor

  - 使用场景: Bulkheading

  - 驱动: 任意的 ``akka.dispatch.ThreadPoolExecutorConfigurator`` 默认为 "thread-pool-executor"

* CallingThreadDispatcher

  - 这个dispatcher仅在当前线程上运行任务。这个dispatcher不会创建人和新线程，但是它可以被同一个actor并发地在不同的线程上使用。参见 :ref:`Java-CallingThreadDispatcher` 了解详情和限制。

  - 共享性质: Unlimited

  - 信箱: 任意, 为每个Actor的每个线程创建一个 (按需)

  - 使用场景: 测试

  - 驱动: 调用者线程 (duh)

更多 dispatcher 配置示例
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

配置一个 ``PinnedDispatcher``:

.. includecode:: ../scala/code/docs/dispatcher/DispatcherDocSpec.scala#my-pinned-dispatcher-config

然后使用它:

.. includecode:: ../java/code/docs/dispatcher/DispatcherDocTest.java#defining-pinned-dispatcher

注意上面 ``my-thread-pool-dispatcher`` 例子中的 ``thread-pool-executor`` 配置是不可用的。 只是因为当使用 ``PinnedDispatcher`` 时每个actor会具有自己的线程池，并且那个线程池仅会拥有一个线程。

注意并不能保证随着时间推移 *相同* 的线程会被使用，因为使用了核心线程池的超时时间，从而在actor闲置时降低资源使用。要一直使用相同的线程，你需要向 ``PinnedDispatcher`` 的配置中添加 ``thread-pool-executor.allow-core-timeout=off`` 。