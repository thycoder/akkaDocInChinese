.. _deployment-scenarios:

###################################
 使用案例和部署场景
###################################

我该如何使用和部署Akka？
==============================

Akka可以通过不同的方式使用：

- 作为一个类库：由web应用使用，放到WEB-INF/lib目录中，或者作为你的classpath上的一个常用Jar包
- 作为一个独立的应用程序：在主类中实例化一个ActorSystem，或使用 :ref:`Microkernel (Scala) <microkernel-scala>` / :ref:`Microkernel (Java) <microkernel-java>`


将Akka用作类库
---------------------

如果你在构建Web应用程序，这是最有可能的情况。通过向你的堆栈添加越来越多的模块，你在类库模式下使用Akka有若干种方式。


将Akka用作一个独立的微内核
----------------------------------------

Akka还可以作为一个独立的微内核来运行。更多详情请参见 :ref:`Microkernel (Scala) <microkernel-scala>` / :ref:`Microkernel (Java) <microkernel-java>` 。
