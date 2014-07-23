.. _extending-akka-java:

########################
 Akka Extensions
########################

如果你想向Akka增加特性，有一种非常优雅，但是强大的机制可以做到。它被称作 Akka Extensions 并且包含2个基本组件: 一个 ``Extension`` 和一个 ``ExtensionId``.

Extension 只会被每个 ``ActorSystem`` 加载一次, 它会被 Akka 管理。 你可以选择按需加载你的Extension，或者在 ``ActorSystem`` 创建时通过Akka 配置而加载. 如何操作的详情如下，在 "从配置读取" 小节.

.. warning::
	
	由于扩展是嵌入Akka自身的一种方式，扩展的实现者需要确保其扩展的线程安全性。


构建一个Extension
=====================

因此让我们创建一个示例扩展，仅让我们统计某事发生的次数。

首先，我们定义我们的 ``Extension`` 应当做什么:

.. includecode:: code/docs/extension/ExtensionDocTest.java
   :include: imports

.. includecode:: code/docs/extension/ExtensionDocTest.java
   :include: extension

然后我们需要为我们的扩展创建一个 ``ExtensionId`` ，从而使我们能够取到它。

.. includecode:: code/docs/extension/ExtensionDocTest.java
   :include: imports

.. includecode:: code/docs/extension/ExtensionDocTest.java
   :include: extensionid

Wicked! 现在我么只需实际使用它: 

.. includecode:: code/docs/extension/ExtensionDocTest.java
   :include: extension-usage

或者从Akka的actor内部使用:

.. includecode:: code/docs/extension/ExtensionDocTest.java
   :include: extension-usage-actor

这是全部的事情!

从配置加载
==========================

为了能够从Akka配置中加载插件，你必须将 ``ExtensionId`` 或 ``ExtensionIdProvider`` 实现的全限定类名添加到你为 ``ActorSystem`` 所提供的配置中的 "akka.extensions" 小节中。

::

    akka {
      extensions = ["docs.extension.ExtensionDocTest.CountExtension"]
    }

实用性
=============

天空就是极限！
顺便问一下，你是否知道Akka的 ``Typed Actors``, ``Serialization`` 和其他特性都以Akka Extensions 的形式实现?

.. _extending-akka-java.settings:

特定于应用程序的设置
-----------------------------

 :ref:`configuration` 可以用于特定于应用程序的设置。一个最佳实践是将这些设置放到扩展中。

示例配置:

.. includecode:: ../scala/code/docs/extension/SettingsExtensionDocSpec.scala
   :include: config

``Extension``:

.. includecode:: code/docs/extension/SettingsExtensionDocTest.java
   :include: imports

.. includecode:: code/docs/extension/SettingsExtensionDocTest.java
   :include: extension,extensionid

使用它:

.. includecode:: code/docs/extension/SettingsExtensionDocTest.java
   :include: extension-usage-actor

