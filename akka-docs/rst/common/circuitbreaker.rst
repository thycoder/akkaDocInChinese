.. _circuit-breaker:

###############
保险丝
###############

==================
它们为何被使用?
==================
保险丝被用于在分布式系统中提供稳定性并且避免级联故障。它们应当在远程系统之间的接口上同合理的超时时间一起使用，从而避免一个单独组件的错误弄坏所有组件。

作为一个示例，我们有一个web应用程序与一个远程的第三方web service交互。让我们假定第三方超额卖出了它们的容量，并且它们的数据库在高负荷之下宕机。假定数据库出错的方式导致将错误传回第三方web service需要很长时间。这进而导致方法调用在很长时间后才会失败。回到我们的web应用，用户会注意到他们的表单提交需要长得多的时间，似乎已经停止响应。用户只能使用他们所知道的办法，就是使用刷新annuity，向他们已经在运行的请求中添加更多的请求。这最终导致web应用资源耗尽而产生故障。这会影响所有用户，甚至是那些使用不依赖于第三方web service的功能的用户。

在web service中引入保险丝会导致请求开始快速出错，使用户知道发生了问题，并且他们不需要刷新请求。这还将错误行为限制到了使用依赖于第三方的功能的用户，其他用户不再受影响，也不会发生资源耗尽的情况。保险丝还可以允许有经验的开发者将标记网站中使用不可用功能的部分，或者当保险丝开启的时候显示一些恰当的缓存内容。

The Akka 库提供了一个保险丝实现，叫做 :class:`akka.pattern.CircuitBreaker` ，其行为如下所述:

=================
他们是做什么的?
=================
* 在常规操作中，保险丝处于 `Closed` 状态:
	* 异常或超过所配置 `callTimeout` 的方法调用会增加错误计数器的值
	* 成功调用会将错误计数器重置为0
	* 当错误计数器达到 `maxFailures` 计数值, 保险丝会切换为 `Open` 状态
* 当处于 `Open` 状态:
	* 所有调用都快速失败，抛出 :class:`CircuitBreakerOpenException`
	* 在所配置的 `resetTimeout` 之后，保险丝进入了 `Half-Open` 状态
* 在 `Half-Open` 状态:
	* 第一次调用尝试会被允许，不再快速失败
	* 所有其他调用都会快速失败并抛出异常，同 `Open` 状态
	* 如果第一次调用成功，则保险丝被置回 `Closed` 状态
	* 如果第一次调用失败，保险丝会重新切换到 `Open` 状态，等待另一个完整的 `resetTimeout`
* 状态切换监听器s: 
	* 可提供针对每个状态入口的回调，通过 `onOpen`, `onClose`, and `onHalfOpen`
	* 这些会在所提供的 :class:`ExecutionContext` 中执行。 

.. image:: ../images/circuit-breaker-states.png

========
示例
========

--------------
初始化
--------------

这里是 :class:`CircuitBreaker` 如何被配置:
  * 5 次最多失败
  * 调用超时时间为 10秒 
  * 重置超时时间为 1分钟

^^^^^^^
Scala
^^^^^^^

.. includecode:: code/docs/circuitbreaker/CircuitBreakerDocSpec.scala
   :include: imports1,circuit-breaker-initialization

^^^^^^^
Java
^^^^^^^

.. includecode:: code/docs/circuitbreaker/DangerousJavaActor.java
   :include: imports1,circuit-breaker-initialization

---------------
调用保护
---------------

这里是  :class:`CircuitBreaker` 如何被用来保护一个异步调用以及一个同步调用:

^^^^^^^
Scala
^^^^^^^

.. includecode:: code/docs/circuitbreaker/CircuitBreakerDocSpec.scala
   :include: circuit-breaker-usage

^^^^^^
Java
^^^^^^

.. includecode:: code/docs/circuitbreaker/DangerousJavaActor.java
   :include: circuit-breaker-usage

.. note::
    
	使用 :class:`CircuitBreaker` 的伴生对象的 `apply` 或 `create` 方法将返回一个 :class:`CircuitBreaker` ，其中回调方法将在调用者线程上执行。这在不必使用异步 :class:`Future` 行为时是很好用的，例如当调用一个仅支持同步的API。
