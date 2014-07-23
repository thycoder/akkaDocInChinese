
.. _multi-jvm-testing:

###################
 多 JVM 测试
###################

支持同时在多个JVM上运行应用 (具有mail方法的对象) 和 ScalaTest 测试。对集成测试多个相互通信的系统非常有用。

设置
=====

多JVM测试时一个sbt插件，可以在此找到 `<http://github.com/typesafehub/sbt-multi-jvm>`_.

你可以通过将下列内容添加到project/plugins.sbt 中，将其添加为一个插件:

.. includecode:: ../../../project/plugins.sbt#sbt-multi-jvm

然后你可以向 ``build.sbt`` 或 ``project/Build.scala`` 中添加多JVM测试，方法是包括 ``MultiJvm`` 设置和配置。 请注意 MultiJvm 测试源代码位于 ``src/multi-jvm/...`` ，而不是 ``src/test/...`` 。

这里是一个例子 ``build.sbt`` 文件，针对 sbt 0.13 ，并使用了 MultiJvm 插件:

.. includecode:: ../../../akka-samples/akka-sample-multi-node-scala/build.sbt

你可以为fork的JVM指定JVM选项::

    jvmOptions in MultiJvm := Seq("-Xmx256M")


运行测试
=============

多JVM任务类似于常规的任务:  ``test``, ``test-only`` 和 ``run``, 但是按照 ``multi-jvm`` 配置来运行。

因此在Akka中，要再 akka-remote 工程中运行所有多JVM测试，请使用 (在sbt命令提示符下):

.. code-block:: none

   akka-remote-tests/multi-jvm:test

或者也可以首先进入 ``akka-remote-tests`` 项目，然后运行测试:

.. code-block:: none

   project akka-remote-tests
   multi-jvm:test

要运行单独的测试请使用 ``test-only``:

.. code-block:: none

   multi-jvm:test-only akka.remote.RandomRoutedRemoteActor

可以列出多个测试名字，来运行多个指定的测试。sbt中的Tab补全使得补全测试名称很容易。

还可以指定 ``test-only`` 的 JVM参数，方法是讲这些选项包含在测试名称和 ``--`` 之后。例如:

.. code-block:: none

    multi-jvm:test-only akka.remote.RandomRoutedRemoteActor -- -Dsome.option=something


创建应用程序测试
==========================

测试是通过一个命名惯例被发现和组合的。MultiJvm 测试源代码位于 ``src/multi-jvm/...`` 。测试的名称遵循以下模式:

.. code-block:: none

    {TestName}MultiJvm{NodeName}

也就是，每个测试的名称中间都包含 ``MultiJvm`` 。 它之前的部分将测试/应用程序分组到单个 ``TestName`` 之下，将被一起运行。后边的部分， ``NodeName``, 是用于区别每个fork的JVM的名字。

因此要创建一个3节点测试，名为 ``Sample`` ，你可以按照如下方式创建三个应用程序 ::

    package sample

    object SampleMultiJvmNode1 {
      def main(args: Array[String]) {
        println("Hello from node 1")
      }
    }

    object SampleMultiJvmNode2 {
      def main(args: Array[String]) {
        println("Hello from node 2")
      }
    }

    object SampleMultiJvmNode3 {
      def main(args: Array[String]) {
        println("Hello from node 3")
      }
    }

当你在sbt命令提示符下调用 ``multi-jvm:run sample.Sample`` ，三个 JVM将被创建，每个节点一个。它将如下所示:

.. code-block:: none

    > multi-jvm:run sample.Sample
    ...
    [info] * sample.Sample
    [JVM-1] Hello from node 1
    [JVM-2] Hello from node 2
    [JVM-3] Hello from node 3
    [success] Total time: ...


修改默认值
=================

你可以通过向你的项目中添加如下配置，来修改多JVM测试的源代码目录名称:

.. code-block:: none

   unmanagedSourceDirectories in MultiJvm <<=
      Seq(baseDirectory(_ / "src/some_directory_here")).join

你可以修改 ``MultiJvm`` 标识符。例如，将其修改为 ``ClusterTest`` ，使用 ``multiJvmMarker`` 设置项:

.. code-block:: none

   multiJvmMarker in MultiJvm := "ClusterTest"


现在你的测试应当命名为 ``{TestName}ClusterTest{NodeName}``.


JVM 实例配置
==================================

你可以为每个JVM指定JVM选项。做法是在测试中创建一个按照这个节点名命名的文件，后缀为 ``.opts`` ，并且将其放到与测试相同的目录中。

例如，要将JVM 选项 ``-Dakka.remote.port=9991`` 和 ``-Xmx256m`` 配置到 ``SampleMultiJvmNode1`` ，让我们创建三个``*.opts`` 文件，并且将选项添加到它们中。使用空格分离多个选项。



ScalaTest
=========

Akka还支持创建ScalaTest 测试，而不仅是应用程序。 为了做到这一点，要使用跟上面相同的命名惯例，但是创建ScalaTest 套件，而不是带有main方法的对象。你需要在classpath中包含ScalaTest。这是一个跟上面类似的例子，但是使用ScalaTest ::

    package sample

    import org.scalatest.WordSpec
    import org.scalatest.matchers.MustMatchers

    class SpecMultiJvmNode1 extends WordSpec with MustMatchers {
      "A node" should {
        "be able to say hello" in {
          val message = "Hello from node 1"
          message must be("Hello from node 1")
        }
      }
    }

    class SpecMultiJvmNode2 extends WordSpec with MustMatchers {
      "A node" should {
        "be able to say hello" in {
          val message = "Hello from node 2"
          message must be("Hello from node 2")
        }
      }
    }

要仅运行这些测试，你应当在sbt命令提示符下调用 ``multi-jvm:test-only sample.Spec`` 。

多节点 Node 扩展
====================

``SbtMultiJvm`` 插件有一些扩展，能够适配 :ref:`experimental <experimental>` 模块 :ref:`multi node testing <multi-node-testing>`,
在相应小节进行了描述。
