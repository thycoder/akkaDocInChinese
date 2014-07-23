
.. highlightlang:: none

.. _building-akka:

###############
 构建 Akka
###############

这个页面描述了如何使用最新代码构建和运行Akka。


获得源代码
===================

Akka 使用 `Git`_ ，并且被托管于 `Github`_.

.. _Git: http://git-scm.com
.. _Github: http://github.com

你首先需要在你的机器上安装git，然后你可以从 http://github.com/akka/akka clone代码仓库

例如 ::

   git clone git://github.com/akka/akka.git

你过你之前已经clone这个代码仓库，那么你可以使用 ``git pull`` 更新你的代码 ::

   git pull origin master


sbt - Simple Build Tool
=======================

Akka使用了优秀的 `sbt`_ 构建系统. 因此你首先要做的事情是下载并安装sbt。你可以再 `sbt setup`_ 文档中读到如何做.

.. _sbt: https://github.com/harrah/xsbt
.. _sbt setup: https://github.com/harrah/xsbt/wiki/Setup

构建Akka时你所需的sbt命令都在下边。如果你想了解git的更多知识，并且在自己的工程中使用它，请阅读 `sbt documentation`_.

.. _sbt documentation: https://github.com/harrah/xsbt/wiki

Akka 的 sbt 构建文件是 ``project/AkkaBuild.scala``.


构建 Akka
=============

首先确保你处于akka 代码目录::

   cd akka


构建
--------

要编译Akka所有的核心模块，请使用 ``compile`` 命令 ::

   sbt compile

你可以使用 ``test`` 命令运行所有测试 ::

   sbt test

如果编译和测试是成功的，那么你已经为工作于最新的Akka开发版本做好了所有准备。


并行执行
------------------

默认情况下测试是顺序执行的。它们可以并发执行，缩短构建时间，如果硬件可以处理增加的内存和cpu使用。添加以下的系统属性到sbt启动脚本，来激活并行执行 ::s

  -Dakka.parallelExecution=true

长期运行和时间敏感的测试
-------------------------------------

默认情况下长期运行的测试（主要是集群测试）和时间敏感的测试（取决于运行所使用的机器性能）是禁用的。你可以通过添加下列标记之一来启用它们 ::

  -Dakka.test.tags.include=long-running
  -Dakka.test.tags.include=timing

或者如果你需要同时启用它们 ::

  -Dakka.test.tags.include=long-running,timing

发布到本地 Ivy 仓库
-------------------------------

如果你想将这些工件部署到你自己的本地Ivy仓库（例如，用于sbt项目），那么请使用 ``publish-local`` 命令::

   sbt publish-local


sbt 交互模式
--------------------

注意在上面的例子中我们调用了 ``sbt compile`` 和 ``sbt test`` 等等, 但是 sbt 还有一种交互式模式。如果你仅运行 ``sbt`` 你会进入交互式的sbt命令提示符，并且能够直接输入命令。这节省了为每条新命令启动一个新的JVM实例的开销，并且更加便捷。

例如，如上构建Akka的过程更常见的是这样做 ::

   % sbt
   [info] Set current project to default (in build file:/.../akka/project/plugins/)
   [info] Set current project to akka (in build file:/.../akka/)
   > compile
   ...
   > test
   ...


sbt 批处理模式
--------------

还有可能将命令组合为一个单独的调用。例如，测试以及发布Akka到本地Ivy仓库可以这样做 ::

   sbt test publish-local


.. _dependencies:

依赖
============

所创建的Ivy依赖解析信息可以通过 ``sbt update`` 来查看 ，也可以在``~/.ivy2/cache`` 找到. 例如，  ``~/.ivy2/cache/com.typesafe.akka-akka-remote-compile.xml`` 文件包含akka-remote 模块的编译依赖。如果你在web浏览器中打开这个文件，你将获得一个易于导航的依赖视图。

Scaladoc 依赖
=====================

Akka 使用 ScalaDoc 为API文档生成类图。这需要安装带有 ``dot`` 命令的Graphviz 软件包，从而避免错误。你可以禁用类图生成，方法是添加一个标记 ``-Dakka.scaladoc.diagrams=false``. 安装 Graphviz 之后，要确保将这个工具集添加到 PATH (当然在 Windows下才需要).
