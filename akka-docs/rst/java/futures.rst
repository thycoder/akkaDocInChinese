.. _futures-java:

Future
===============

简介
------------

在Scala标准库中,  `Future <http://en.wikipedia.org/wiki/Futures_and_promises>`_ 是一种数据结构，用于获取一些并发操作的结果。结果可以同步（阻塞）或异步（非阻塞）地访问。为了能够在Java中使用它，Akka在 ``akka.dispatch.Futures`` 中提供了一个对java友好的接口。

Execution Context
------------------

为了执行回调函数和操作，Future需要在 ``ExecutionContext`` 上调用一些方法，它非常类似于 ``java.util.concurrent.Executor``. 如果在作用域中有一个n ``ActorSystem`` ，他会将其默认dispatcher用作 ``ExecutionContext``, 或者你也可以使用 ``ExecutionContexts`` 类所提供的工厂方法，来包装 ``Executors`` 和 ``ExecutorService``, 甚至可以自己创建。

.. includecode:: code/docs/future/FutureDocTest.java
   :include: imports1,imports7

.. includecode:: code/docs/future/FutureDocTest.java
   :include: diy-execution-context

跟Actor一起使用
---------------

通常有两种方式可以从一个 ``UntypedActor`` 取得回复: 第一种是通过发送消息 (``actorRef.tell(msg, sender)``),
这仅当原始发送者是一个 ``UntypedActor`` 时可行) 而第二种是通过一个 ``Future``。

使用 ``ActorRef`` 的 ``ask`` 方法来发送一个消息会返回一个 ``Future``。
等待获取实际结果的最简单方法是:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: imports1

.. includecode:: code/docs/future/FutureDocTest.java
   :include: ask-blocking

这会导致当前线程阻塞，并且等待 ``UntypedActor`` 来 'complete' 这个 ``Future`` ，使用其回复。
阻塞是不被鼓励的，因为它会导致性能问题。阻塞操作位于 ``Await.result`` 和 ``Await.ready`` ，从而使得阻塞发生的地方很容易看出。
阻塞的替代方案在这个文档中进行了更加深入的探讨。
还要注意， ``UntypedActor`` 所返回的 ``Future`` 是一个 ``Future<Object>`` ，因为 ``UntypedActor`` 是动态的。这是上面的例子中强制转换为 ``String`` 的原因。

.. warning::
   
   ``Await.result`` 和 ``Await.ready`` 被提供，是为了处理你 **必须** 阻塞的例外情形的，一个经验法则是，仅当你了解为何 **必须** 阻塞时才使用它们。对于所有其他情况，使用下述的异步组合。

为了将一个 ``Future`` 的结果发送到一个 ``Actor``, 你可以使用 ``pipe`` 结构:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: pipe-to

直接使用
------------

Akka中一个常见的使用场景是，并发地执行一些计算，而不需要 ``UntypedActor`` 的特殊工具。如果你发现自己仅仅为了并发地执行计算而创建了一个  ``UntypedActor`` 池，那么还有更加简单（和快捷）的方法:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: imports2

.. includecode:: code/docs/future/FutureDocTest.java
   :include: future-eval

在上述代码中，传递到 ``future`` 的代码块会被默认的 ``Dispatcher`` 执行，使用代码块的返回值来完成这个 ``Future`` (在这种情况下，结果将是字符串 "HelloWorld")。不像 ``UntypedActor`` 所返回的 ``Future`` ，这个 ``Future`` 具有正确的类型，并且我们还避免了管理 ``UntypedActor`` 的开销。

你还可以使用 ``Futures`` 类创建已完成的Future, 这可以是成功的:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: successful

也可以是失败的:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: failed

还可能创建空的 ``Promise``, 后续再填充，并且获得相应的 ``Future``:

.. includecode:: code/docs/future/FutureDocTest.java#promise

对于这些例子， ``PrintResult`` 的定义如下:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: print-result

函数式的 Future
------------------

Scala 的 ``Future`` 具有一些 monad 方法，非常类似于 ``Scala``'s 集合中所用的那些方法。这些方法允许你创建结果可以在其中穿行的 '管道' 或 '流' that the result will travel through.

Future 是一个 Monad
^^^^^^^^^^^^^^^^^

第一个用于操作 ``Future`` 的函数式方法是 ``map``. 这个方法接收一个 ``Mapper`` ，它在 ``Future`` 的结果上执行一些操作，并且返回一个新的结果。 ``map`` 方法的返回值是另一个 ``Future`` ，它会包含新的结果:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: imports2

.. includecode:: code/docs/future/FutureDocTest.java
   :include: map

在这个例子中，我们在 ``Future`` 中将两个字符串连接起来。我们不是 f1 完成，使用 ``map`` 方法应用我们计算字符串长度的函数。现在我们拥有了另一个 ``Future``, f2, 它最终将包含一个 ``Integer`` 。 当我们原始的 ``Future``, f1, 完成时, 它还会应用我们的函数，并且使用它的结果完成第二个 ``Future``。 当我们最终 ``get`` 结果时，它将包含数字 10。我们原始的 ``Future`` 仍然包含字符串 "HelloWorld" 并且不受 ``map`` 影响。

使用这些方法时需要注意：如果当这些方法中的某个方法被调用时， ``Future`` 仍在被处理，那么实际做这些事情的，将是完成Future的线程。但如果 ``Future`` 已经被完成，它将在当前线程上运行。例如:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: map2

原始的 ``Future`` 现在将至少耗费  0.1 秒来执行完毕，这意味着当我们调用 ``map`` 时它仍在被处理。我们所提供的函数会被存储在 ``Future`` 中，并且稍后当结果准备好时，会被dispatcher自动执行。

如果我们做相反的事情：

.. includecode:: code/docs/future/FutureDocTest.java
   :include: map3

我们的小字符串在0.1秒的睡眠完成之前早已被处理。因此，disp会继续处理其他待处理的消息，并且不再为我们计算字符串长度，相反，它会在当前线程内被计算，正如我们不使用 ``Future`` 时那样。

通常这会按照它的意思很好地工作，运行一个快速的函数只有极少的开销。如果这个函数有可能耗费大量时间来处理，更好的方式是让它并发地执行，而对于这种情况我们会使用 ``flatMap``:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: flat-map

限制我们第二个 ``Future`` 也被并发地执行。这个技术还可以被用于将若干Future的结果在一个单独的计算中组合，这会在后续小节中进行更好的解释。
如果你需要进行带条件的传播，你可以使用 ``filter``:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: filter

组合 Future
^^^^^^^^^^^^^^^^^

经常会需要将不同的Future相互组合，下面是一些关于这件事如何以一种非阻塞的风格来进行的例子。

.. includecode:: code/docs/future/FutureDocTest.java
   :include: imports3

.. includecode:: code/docs/future/FutureDocTest.java
   :include: sequence

为了更好地解释这个例子中所发生的事情， ``Future.sequence`` 接收一个 ``Iterable<Future<Integer>>`` 并且将其变为 ``Future<Iterable<Integer>>``. 然后我们就可以使用 ``map`` 来直接处理 ``Iterable<Integer>`` ，并且我们聚合了 ``Iterable`` 的和。

``traverse`` 方法类似于 ``sequence``, 但是它接收一系列 ``A`` 并且应用一个从 ``A`` to ``Future<B>`` 的函数，并且返回一个 ``Future<Iterable<B>>`` ,允许对这个序列进行并行的 ``map`` , 如果你使用 ``Futures.future`` 来创建 ``Future`` 的话。

.. includecode:: code/docs/future/FutureDocTest.java
   :include: imports4

.. includecode:: code/docs/future/FutureDocTest.java
   :include: traverse

这就是那么简单！

然后有一个方法叫做 ``fold`` ，接收一个起始值，一系列 ``Future`` ， 以及 一个从起始值类型，一个超时和Future的类型返回一些与起始值类型相同的东西，然后将这个函数应用到future序列中的所有元素，非阻塞地，执行将在最后一个Future完成之后开始。

.. includecode:: code/docs/future/FutureDocTest.java
   :include: imports5

.. includecode:: code/docs/future/FutureDocTest.java
   :include: fold

这就是所需的一切！

如果传入 ``fold`` 的序列是空的，它会返回起始值，在上面的情况中，这将是空字符串。在某些情况下你没有起始值，并且你能够使用序列中第一个完成的 ``Future`` 作为起始值，你可以使用 ``reduce``, 它的工作方式为:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: imports6

.. includecode:: code/docs/future/FutureDocTest.java
   :include: reduce

跟 ``fold`` 相同，执行将在最后一个Future完成后开始，你也可以将其变为并发，通过将你的future分块称为子序列并且reduce它们，然后将reduce的结果再次reduce。

这仅仅是你所能做的事情的一个示例。

回调
---------

有事你只想监听一个 ``Future`` 的完成，并且不是通过创建一个新的Future来对此做出反应，而是通过副作用。
为此Scala支持 ``onComplete``, ``onSuccess`` 和 ``onFailure``, 其中后两种是第一种的特殊化。

.. includecode:: code/docs/future/FutureDocTest.java
   :include: onSuccess

.. includecode:: code/docs/future/FutureDocTest.java
   :include: onFailure

.. includecode:: code/docs/future/FutureDocTest.java
   :include: onComplete

定序
--------

由于回调以任何顺序被执行，并且有可能并行地执行，当你需要对操作确定顺序时会很复杂。但是有个办法！它的名字叫做 ``andThen``, 并且它会创建一个新的 ``Future`` 包含指定的回调, 这个 ``Future`` 会具有与调用方法的 ``Future`` 具有相同的结果，它允许类似下面的示例中的顺序：
which allows for ordering like in the following sample:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: and-then

辅助方法
-----------------

``Future`` ``fallbackTo`` 将2个Future组合为一个新的 ``Future``, 并且将持有第二个 ``Future`` 的成功值，如果第一个 ``Future`` 失败的话。

.. includecode:: code/docs/future/FutureDocTest.java
   :include: fallback-to

你还可以将两个Future组合为一个新的 ``Future`` ，它持有包含两个Future的成功结果的一个元组，这要使用 ``zip`` 操作.

.. includecode:: code/docs/future/FutureDocTest.java
   :include: zip

异常
----------

由于 ``Future`` 的结果的创建跟程序的其他部分是并发的, 异常必须用不同的方式处理。一个 ``UntypedActor`` 或 dispatcher 是否完成 ``Future`` 并不重要, 如果一个  ``Exception`` 被捕获， ``Future`` 将包含它，而不是包含一个有效的结果。 如果 ``Future`` 确实包含一个 ``Exception`` ，调用 ``Await.result`` 会导致它再次被抛出，从而使得它能够被正确地处理。 

还可能在处理 ``Exception`` 时返回一个不同的结果。
这是通过 ``recover`` 方法来做到的. 例如:

.. includecode:: code/docs/future/FutureDocTest.java
   :include: recover

在这个例子中，如果actor回复一个 ``akka.actor.Status.Failure`` ，其中包含一个 ``ArithmeticException``, 我们的 ``Future`` 会具有一个结果为0。  ``recover`` method 的工作方式跟标准的 try/catch 代码块非常类似，因此多个 ``Exception`` 可以按照这种方式进行处理, 并且如果一个 ``Exception`` 没有按照这样的方式处理，它将表现为跟没有使用 ``recover`` 方法一样。

你还可以使用 ``recoverWith`` 方法，它跟 ``recover`` 的关系类似于 ``flatMap`` 和 ``map`` 的关系，并且用法如下；

.. includecode:: code/docs/future/FutureDocTest.java
   :include: try-recover

After
-----

``akka.pattern.Patterns.after`` 使得很容易使用一个值完成一个 ``Future`` 或在超时之后使用异常来完成它。

.. includecode:: code/docs/future/FutureDocTest.java
   :include: imports8

.. includecode:: code/docs/future/FutureDocTest.java
   :include: after
