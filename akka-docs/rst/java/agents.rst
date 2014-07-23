.. _agents-java:

########
 Agent
########

Akka中的Agent 收到了 `agents in Clojure`_  的启发。

.. _agents in Clojure: http://clojure.org/agents

Agent提供了单独位置的异步修改。Agent在它们的一生中都被绑定到一个单独的存储位置，并且只允许读取那个位置的修改作为一个动作的结果而发生。更新动作是一个可以被异步应用到Agent的状态的函数，并且其返回值会成为Agent的新状态。Agent的状态应当是不可变对象。

虽然对Agent的更新是异步的，但Agent的状态总是能够由任意线程立即读取（使用 ``get`` ）,不需要任何消息。

Agent是响应式的（reactive）. 所有Agent的更新操作会在一个 ``ExecutionContext`` 的多个线程中交替执行。 在任何时间点，每个Agent最多有一个 ``send`` 动作在执行。从另一个线程调度到一个agent的动作会按照它们被发送的顺序发送，可能与其他线程调度到同一个agent的动作交替进行。


创建 Agent
===============

Agents 的创建需要通过  ``new Agent<ValueType>(value, executionContext)`` – 传入Agent的初始值，并且提供一个 ``ExecutionContext`` 供它使用:

.. includecode:: code/docs/agent/AgentDocTest.java
   :include: import-agent,create
   :language: java

读取一个Agent的值
========================

Agent可以被解引用（你可以获得一个Agent的值），方法是使用 ``get()`` 来调用Agent，像这样：

.. includecode:: code/docs/agent/AgentDocTest.java#read-get
   :language: java

读取Agent的当前值不涉及任何消息传递，并且立即发生。因此虽然对Agent的更新是异步的，读取Agent的状态却是同步的。

你还可获得Agent的值的一个 ``Future`` ，在当前排队的更新都完成之后，它才会被完成。

.. includecode:: code/docs/agent/AgentDocTest.java
   :include: import-future,read-future
   :language: java

参见 :ref:`futures-java` ，了解关于 ``Futures`` 的更多信息。

更新 Agent (发送 & 修改)
==============================

你对Agent的更新需要通过发送一个函数(``akka.dispatch.Mapper``) ，它将当前的值进行转换，或者仅发送一个新的值。Agent会异步地应用新的值或函数。更新以一种激发并遗忘的方式来进行，并且你得到的保证仅限于它会被应用。不能保证这个更新何时被应用，但是从一个单独的县城向一个Agent的调度会按照顺序发生。你通过调用 ``send`` 函数来应用一个值或函数。

.. includecode:: code/docs/agent/AgentDocTest.java
   :include: import-function,send
   :language: java

你还可以调度一个函数用于更新内部状态，但是在它自己的线程上进行。这不能使用响应式的线程池，并且可以用于长期运行或阻塞的操作。你需要使用 ``sendOff`` 来作这件事。使用 ``sendOff`` 或 ``send`` 所进行的调度仍然会按顺序执行。

.. includecode:: code/docs/agent/AgentDocTest.java
   :include: import-function,send-off
   :language: java

所有 ``send`` 方法还有一个对应的 ``alter`` 方法，返回一个 ``Future``.
参见 :ref:`futures-java` 了解关于 ``Future`` 的更多信息。

.. includecode:: code/docs/agent/AgentDocTest.java
   :include: import-future,import-function,alter
   :language: java

.. includecode:: code/docs/agent/AgentDocTest.java
   :include: import-future,import-function,alter-off
   :language: java

配置
=============

agent模块有一些配置属性，请参见 :ref:`reference configuration <config-akka-agent>`.

废弃的带事务Agents
===============================

参与外围STM事务的Agent在 2.3 版本中是一个废弃的特性。

如果一个Agent在外围的 ``Scala STM transaction`` 内使用, 那么它将参加那个事务。如果你在一个事务中向一个Agent发送消息，那么对这个Agent的调度将会被暂停，直到事务提交，并且如果事务中止则丢弃这些调度。
