.. _Duration:

########
Duration
########

Duration 在 Akka 库中到处被使用,  然而这个概念是由一个特殊的数据类型表示的，即 :class:`scala.concurrent.duration.Duration`.
这个类的值可以表示无限 (:obj:`Duration.Inf`,
:obj:`Duration.MinusInf`) 或有限的时间间隔，或者 :obj:`Duration.Undefined`.

有限 vs. 无限
===================

由于试图将一个无限的时间间隔转换到一个具体的时间单位如秒，会抛出一个异常，存在不同的类型可用于在编译时区别这两种时间间隔:

* :class:`FiniteDuration` 保证是有限的, 调用 :meth:`toNanos` 及其伙伴是安全的
* :class:`Duration` 可以是有限或者无限的, 因此这个类型应当在是否无限无所谓的情况下使用;这是 :class:`FiniteDuration` 的一个超类

Scala
=====

在Scala中，时间间隔可以使用一个微型DSL来构建，并且支持所有预期的算术操作:

.. includecode:: code/docs/duration/Sample.scala#dsl

.. note::
   
   你可以省略这些点，如果这个表达式的边界是清晰的(例如在括号中，或者在参数列表中)，但是推荐当时间单元是一行的最后一个符号时使用点，否则分号推导可能出错，取决于下一行的开头是什么。

Java
====

Java 提供的语法糖更少，所以你必须将这些操作拼写为方法调用作为替代:

.. includecode:: code/docs/duration/Java.java#import
.. includecode:: code/docs/duration/Java.java#dsl

Deadline
========

Duration有个兄弟名叫 :class:`Deadline`, 这个类持有一个绝对时间点的表示，并且支持通过计算现在和这个截止时间之间的差异，来获得一个时间间隔。这在你想保持一个全局的截止时间而不想处理记录的杂务时非常有用。自己传递时间:

.. includecode:: code/docs/duration/Sample.scala#deadline

在Java中你从时间间隔创建这些:
In Java you create these from durations:

.. includecode:: code/docs/duration/Java.java#deadline
