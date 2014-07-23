
.. _scheduler-java:

##################
 Scheduler
##################

有时会出现让事情在未来发生的需求，那么你去哪里查找方法？不需要在 ``ActorSystem`` 之外查找! 在其中你可以找到 :meth:`scheduler` 方法，它返回一个  :class:`akka.actor.Scheduler` 实例，这个实例对于每个ActorSystem是唯一的，并且在内部被用于调度一些事情，使之在指定的时间点发生。

你可以调度向actor发送消息，或者任务的执行(函数 or Runnable).  你将得回一个 ``Cancellable`` ，你可以在其上调用 :meth:`cancel` 来取消所调度的事情的执行。

.. warning::
    Akka所使用的 ``Scheduler`` 的默认实现是基于按照固定的调度方式被清空的任务桶。它并不在精确的时间执行任务，但是在每次滴答时，它会执行所有到期（或过期）任务。默认Scheduler的精度可通过 ``akka.scheduler.tick-duration`` 配置属性来进行修改。

一些例子
-------------

调度再50ms吼向testActor发送一个 "foo" 消息:

.. includecode:: code/docs/actor/SchedulerDocTest.java
   :include: imports1

.. includecode:: code/docs/actor/SchedulerDocTest.java
   :include: schedule-one-off-message
调度一个Runnable,它将当前时间发送给testActor，在50ms后执行:

.. includecode:: code/docs/actor/SchedulerDocTest.java
   :include: schedule-one-off-thunk

.. warning::
    
	如果你调度Runnable实例，你应当特别当心，不要传递或捕获不稳定的引用。在实践中，这意味着你不应当在Runnable内部调用外围Actor的方法。如果你需要调度一次方法调用，你最好使用 ``schedule()`` 的一个接收一条消息和一个 ``ActorRef`` 的变化形式，来调度一条发往自身的消息(包含必要的参数)，然后当接收到消息时再调用方法。

调度在0ms后每50ms重复地向
Schedule to send the  ``tickActor`` 发送"Tick"消息:

.. includecode:: code/docs/actor/SchedulerDocTest.java
   :include: imports1,imports2

.. includecode:: code/docs/actor/SchedulerDocTest.java
   :include: schedule-recurring

来自 ``akka.actor.ActorSystem``
-------------------------------

.. includecode:: ../../../akka-actor/src/main/scala/akka/actor/ActorSystem.scala
   :include: scheduler

面向实现者的Scheduler接口
----------------------------------------

实际的scheduler实现是在 :class:`ActorSystem` 启动时通过反射加载的, 这意味着可以使用 ``akka.scheduler.implementation``配置属性提供一个不同的值。被引用的类必须实现以下接口:

.. includecode:: ../../../akka-actor/src/main/java/akka/actor/AbstractScheduler.java
   :include: scheduler

Cancellable 接口
-------------------------

调度一个任务会产生一个 :class:`Cancellable` (或抛出一个 :class:`IllegalStateException` 如果在scheduler 关闭后尝试的话)。这允许你取消已经被调度执行的任务。

.. warning::
  
  这不会中断任务执行，如果它已经开始的话。检查 ``cancel`` 的返回值，可以检测被调度的任务已经取消，还是最终会执行。

.. includecode:: ../../../akka-actor/src/main/scala/akka/actor/Scheduler.scala
   :include: cancellable

