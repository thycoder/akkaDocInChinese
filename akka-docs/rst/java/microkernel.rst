
.. _microkernel-java:

Microkernel
==================

Akka Microkernel 的目的是提供一种打包机制，让你能够将Akka应用独立地发布，无需运行于一个Java应用服务器或通过创建一个启动脚本来手动运行。

Akka Microkernel 包含在Akka 相关的下载中，网址为 `downloads`_.

.. _downloads: http://akka.io/downloads

要想使用微内核运行一个应用，你需要创建一个Bootable 类，处理应用程序的启动和关闭。一个例子包含于下面。

将你的应用程序jar放到 ``deploy`` 目录，额外的依赖放到 ``lib`` 目录，来让它们被自动加载并置于classpath中。

要启动这个内核，你需要使用 ``bin`` 目录中的脚本，传入你的引用程序中的引导类。

要启动脚本，首先要再classpath中加入 ``config`` 目录，紧随其后是 ``lib/*`` 。它会运行java，使用主类  ``akka.kernel.Main`` 以及所提供的 Bootable 类作为参数。

示例命令 (在一个基于unix的系统上):

.. code-block:: none

   bin/akka sample.kernel.hello.HelloKernel

使用 ``Ctrl-C`` 来中断并退出微内核。

在一台 Windows 机器上，你也可以使用 bin/akka.bat 脚本.

Hello Kernel 示例中的代码 (参见 ``HelloKernel`` 类 ，作为一个创建 Bootable 的例子):

.. includecode:: ../../../akka-samples/akka-sample-hello-kernel/src/main/java/sample/kernel/hello/java/HelloKernel.java


