##########################
必不可少的Hello World
##########################

向控制台打印一句著名的欢迎文字的困难任务的基于actor的版本在一个 `Typesafe Activator <http://www.typesafe.com/platform/getstarted>`_ 教程名为 `Akka Main in Java <http://www.typesafe.com/activator/template/akka-sample-main-java>`_ 中进行了介绍。

这个教程阐述了通用的启动类 :class:`akka.Main` ，它只预期一个命令行参数：应用程序主actor的类名。 main方法然后会创建运行这些actor所需的基础设施，启动给定的主actor，并且安排整个应用程序在主actor终止时立即关闭。


同一个问题域还有另一个 `Typesafe Activator <http://www.typesafe.com/platform/getstarted>`_ 教程，名为 `Hello Akka! <http://www.typesafe.com/activator/template/hello-akka>`_ 。它更深入地描述了Akka的基础知识。


