.. _fsm-java:

###########################################
构建有限状态自动机actor
###########################################

概述
========

对FSM (有限状态自动机) 模式最好的描述是 `Erlang design
principles
<http://www.erlang.org/documentation/doc-4.8.2/doc/design_principles/fsm.html>`_.
简言之，他可以看做一组关系,形式如下:

  **State(S) x Event(E) -> Actions (A), State(S')**

这些关系的含义是：

  *如果我们处于状态S，并且发生了事件E，我们应当执行动作A，并且转移到状态S'。*

虽然Scala编程语言允许简洁描述一个精妙的内部DSL(领域特定语言)用于阐述有限状态自动机（参见 :ref:`fsm-scala`), Java 的繁琐语法并不能很好地适应这种方式。本章描述了通过自我约束来实现同样的分离关注点的方式。

状态应当如何被处理
===========================

所有被FSM actor的实现所引用的可变字段（或传递可变的数据结构）应当在一处被收集，并且仅适用一组很小的定义良好的方法集合来修改。实现这一点的一种方式是在一个超类中组装所有可变状态，将其设置为私有，并且提供受保护的方法用于修改它。

.. includecode:: code/docs/actor/FSMDocTest.java#imports-data

.. includecode:: code/docs/actor/FSMDocTest.java#base

这个方式的好处是，状态修改可以在一个集中的地方来进行，使得向FSM的机器中添加内容时不可能忘记插入响应状态转移的代码。

Message Buncher 示例
=======================

上面所示的超类被设计为支持一个类似于Scala FSM文档中的示例：一个actor接收消息并将消息排队，批量发送给一个可配置的目标actor。涉及的消息有:

.. includecode:: code/docs/actor/FSMDocTest.java#data

这个actor只有两种状态 ``IDLE`` 和 ``ACTIVE``, 使得在由基类所派生的具体actor中处理状态特别简洁：

.. includecode:: code/docs/actor/FSMDocTest.java#imports-actor

.. includecode:: code/docs/actor/FSMDocTest.java#actor

此处的技巧是，分离出公共的功能如 :meth:`whenUnhandled`
何 :meth:`transition` ，从而获得一些定义良好的点，用于响应修改或插入日志。

以State为中心 vs. 以事件为中心
===============================

在上面的例子中，状态和事件的主观复杂度基本相当，使得选择主要的dispatcher变为一个品味问题；在这个例子中，选择了基于状态的调度。屈居于可能的状态和事件矩阵多么均匀，更实际的做法是首先处理不同的事件，然后在第二层分辨状态。一个例子是一个具有大量内部状态但只处理极少不同事件的状态机。
