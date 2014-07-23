.. _developer_guidelines:

开发者指导方针
====================

.. note::

    请首先阅读 `The Akka Contributor Guidelines <https://github.com/akka/akka/blob/master/CONTRIBUTING.md>`_ .

代码风格
----------

Akka代码风格遵循 `Scala Style Guide <http://docs.scala-lang.org/style/>`_ 。唯一的例外是块注释的样式:

.. code-block:: scala

  /**
    * Style mandated by "Scala Style Guide"
    */

  /**
   * Style adopted in the Akka codebase
   */

作为构建过程的一部分，Akka 使用 ``Scalariform`` 来格式化源代码。因此尽管修改代码，然后运行 ``sbt compile`` ，它会重新按照Akka标准来重新格式化代码。

过程
-------

* 确保你签署了Akka CLA, 如果没有的话, `线上签署 <http://www.typesafe.com/contribute/cla>`_.
* 挑选一个 ticket, 如果你的工作没有ticket，则首先创建一个。
* 开始在一个特性分支上工作，命名类似于 ``wip-<ticket number>-<descriptive name>-<your username>``.
* 当你完成后，创建一个指向目标分支的 GitHub Pull-Request 并且向 Akka Mailing List 发邮件，表示你希望这些代码被评审。
* 当评审取得一致后，某个Akka Core Team成员会将其合并。

提交消息
---------------

当创建公共提交并书写提交消息时，请遵循这些指导方针.

1. 如果你的工作跨越了多次本地提交 (作为一个例子; 如果你在一个话题分支上工作时进行安全点提交，或者长时间地工作于一个分支上，进行代码合并或重新分配等等。)那么请 **不要** 全部提交，而是通过将这些提交压缩为一次单独的提交来重写提交历史，并且编写很好的提交消息（如下所讨论）。这是一个关于如何做到的很好的文章: `http://sandofsky.com/blog/git-workflow.html <http://sandofsky.com/blog/git-workflow.html>`_. 每次提交都应当能够单独被使用，挑选最优，等等。

2. 第一行应当是一个描述性的句子，描述这次提交做了什么。仅阅读这一行就应当能够完全明白这次提交所做的事情。 **不** 应当仅仅列出ticket号码， 键入 "小修改" 或类似的做法。在第一行的末尾，在ticket号码中包含引用，加上#前缀。 如果这次提交是一个 **小** 修改，那么到此为止。如果不是，转到第3步。

3. 紧跟这单独一行描述之后，应当是一个空行，空行后是这次提交详情的罗列。

例如 ::

    Completed replication over BookKeeper based transaction log. Fixes #XXX

      * Details 1
      * Details 2
      * Details 3

测试
-------

所有签入的代码 **应当** 有测试. 所有测试使用 ``ScalaTest`` 和 ``ScalaCheck`` 进行。

* 将测试命名为 **Test.scala** ，如果它们不依赖于任何外部的东西。这会满足surefire。
* 将测试命名为 **Spec.scala** ，如果它们具有外部依赖。

Actor TestKit
^^^^^^^^^^^^^

有一个很有用的测试工具集可用于测试actor: `akka.util.TestKit <@github@/akka-testkit/src/main/scala/akka/testkit/TestKit.scala>`_. 它允许关于接收回复及其时间消耗的断言，在 :ref:`akka-testkit` 模块中包含更多文档。

多JVM 测试
^^^^^^^^^^^^^^^^^

例子重包含一个sbt的trait，用于多JVM测试，它会fork JVM用于多节点测试。可支持运行应用程序(具有main方法的对象)，也支持ScalaTest测试。

NetworkFailureTest
^^^^^^^^^^^^^^^^^^

你可以使用 'NetworkFailureTest' trait 来测试网络故障。
