.. _supervision:

监管和监控
==========================

本章简单地介绍监管背后的概念，所提供的原语，及其语义。关于如何转换为代码，请参阅Scala和Java API中的相应章节。

.. _supervision-directives:

监管的含义
----------------------

如 :ref:`actor-systems` 中所述，监管描述了actor间的一种依赖关系：监管者将任务委托给附属的actor并对下属的失败状况进行响应。当一个下属检测到故障（即抛出一个异常），它会将自己以及所有的下属挂起，然后向自己的监管者发送一条报告故障的消息。取决于所监管的工作以及故障性质，监管者有以下4种选项： 

#. 让下属继续执行，保持其所积累的内部状态
#. 重启下属, 清除其所积累的内部状态
#. 永久终止下属
#. 向上升级此故障，进而升级发生故障这件事情本身。

重要的一点是，要始终将一个actor视为一个监管层次结构的一部分，这解释了第四个选项存在的意义（因为一个监管者同时也是另一个更高层级的监管者的下属），并且在前三种选项中也存在隐含的语义：让继续执行actor会让其下属也继续执行，重启一个actor也导致其下属被重启（但是查看后面的内容可了解更多信息），类似地，终止一个actor会终止它所有的下属。值得注意的一点是，:class:`Actor` 类中的 :meth:`preRestart` 钩子方法的默认行为是，重启actor之前会终止其所有下属，但这种行为可以被重写；递归重启操作会应用于钩子方法执行之后所剩余的所有子actor。

每个监管者都配置了一个函数，它将所有可能的失败原因（也就是异常）翻译为以上给出的四种选项之一；注意，这个函数并不将失败actor的标识作为输入。很容易可以想出一些例子结构，在其中这种方式看起来不够灵活，例如，希望对不同的下属应用不同的策略。在这一点上，我们一定要明白，监管是为了形成一个递归的故障处理结构。如果你试图在某个层级做太多事情，这个层次会变得难以理解，这时我们推荐的方法是增加一个监管层级。

Akka实现的是一种叫做“父监管”的形式。Actor只能由其它的actor创建，而顶层的actor是由类库来提供的——每个被创建的actor都由它的父actor所监管。这种限制使得actor的监管层次结构隐式地形成，并鼓励合理的设计决策。应当注意的是，这也同时保证了actor不会成为孤儿，或者从外部连接到监管者，否则就有可能不知不觉地连接它们。此外，这会产生actor（子树）应用的一个自然而干净的关闭过程。

.. warning::

    与监管相关的父子通信通过特殊的消息来进行，这些消息有自己的信箱，与用户信息是隔离的。这暗示了与监管相关的消息相对于普通消息没有确定的顺序。总而言之，用户不能够影响正常消息和故障通知的顺序。如需细节和示例，请参见:ref:`message-ordering` 小节。

.. _toplevel-supervisors:

顶层监管者
-------------------------

.. image:: guardians.png
   :align: center
   :width: 360

Actor系统创建的过程中会启动至少三个actor，如上图所示。关于actor path的影响，如需了解更多，请参见 :ref:`toplevel-paths`.

.. _user-guardian:

``/user``: 守护Actor
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

最经常与之打交道的actor是所有用户创建的actor的父actor，名为``"/user"``的守护actor。 使用 ``system.actorOf()`` 所创建的actor是此actor的子actor. 这意味着，当这个守护actor终止时, 系统中所有普通actor也都会被关闭。这还意味着，这个守护actor的监管策略决定顶层普通actor如何被监管。自Akka 2.1起，这有可能通过设置 ``akka.actor.guardian-supervisor-strategy`` 来进行配置,此配置项需要设置 一个:class:`SupervisorStrategyConfigurator`子类的完整类名。当守护actor升级一个错误时，根守护Actor的响应将是终止此守护Actor，这实际上会关闭整个actor系统。

``/system``: 系统守护Actor
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

这一特殊的守护actor被引入的目的，是实现一个有序的关闭序列，其中当所有普通actor关闭的过程中，日志保持可用，尽管日志本身也是用actor实现的。这是通过让系统守护actor观察用户守护actor，并且在收到:class:`Terminated` 消息时开启自身的关闭过程。顶层的系统actor会在收到:class:`ActorInitializationException` 和 :class:`ActorKilledException`时终止有问题的子actor，收到除此之外所有的:class:`Exception`则无限地重启。所有其他的throwable会沿着监管层次向上传递，从而关闭整个actor系统。

``/``: 根守护Actor
^^^^^^^^^^^^^^^^^^^^^^^^

根守护Actor是一切所谓的“顶层”actor的祖父，并且使用 ``SupervisorStrategy.stoppingStrategy`` 策略监控:ref:`toplevel-paths`中提到的所有特殊actor，该策略的目的是，在收到任何类型的:class:`Exception`时终止相应的子Actor。所有其他的throwable会沿着监管层次上报… 但是向谁? 既然所有真实actor都有一个监管者，那么根守卫Actor不应当是真实actor。而且由于这意味着它“气泡之外”，因此它被称为“气泡行者(bubble-walker)” 这是一个人工合成的 :class:`ActorRef`，它实际上一遇到子actor出现麻烦的情况，就关闭其所有的子actor，并且当根守护actor完全终止（所有子actor递归终止）时，立即将actor系统的 ``isTerminated`` 状态设置为 ``true``。

.. _supervision-restart:

重启的含义
---------------------

当面对一个处理特定消息时出错的actor时，失败的原因分成三类:

* 接收特定消息时的系统 (即编程) 错误。
* 处理消息时所使用的外部资源出现 (瞬时) 失败。
* actor内部状态错误。

除非这一错误能够被明确识别，第三个原因不能被排除，通过这一点可以得出结论，内部状态需要清理掉。如果监管者确定，它其他的子actor或其本身不受此错误的影响——例如，由于有意地使用错误内核模式——因此它最好重启其子actor。这是通过创建底层 :class:`Actor`类的一个全新 实例，并在子actor的:class:`ActorRef`之内用全新的实例替换出错的实例；为了能做到这一点，是将actor封装在特殊的actor引用之内的原因之一。新的Actor然后继续处理其邮箱，这意味着重启actor在其自身之外不可见，不过存在一个例外，就是错误发生期间的消息不会被重新处理。

以下是重启过程中所发生事件的精确次序：

#. 挂起actor（这意味着在继续之前，它不会处理正常消息），并且递归地挂起所有的子actor
#. 调用旧实例的:meth:`preRestart`钩子（默认向所有子actor发送终止请求，并且调用:meth:`postStop`）
#. 等待所有在:meth:`preRestart`期间被请求终止的子actor实际终止。这个——跟所有actor操作类似——是非阻塞的，最后一个子actor终止的通知会导致程序推进到下一步
#. 再次调用一开始提供的工厂来创建新的actor实例
#. 对新实例调用:meth:`postRestart`(它默认还会调用:meth:`preStart`)
#. 恢复运行新的actor
#. 向第三步未杀死的子actor发送重启请求；重启子actor将递归地遵循同样的过程， 从第2步开始。
#. 继续执行actor

生命周期监控的含义
-------------------------------

.. note::

    生命周期监控在Akka中通常被称为``DeathWatch``

与上文所描述的父子actor之间的特殊关系形成鲜明对比的是，每个actor都能够监控任意actor。由于actor在创建时就完全是活着的，而重启在受影响的监管者之外不可见，因此唯一能够监控的状态变化就是从活着到死去。监控因此被用于将一个actor关联到另一个actor，从而使得它能够在对方终止时做出反应，相反地，监管是对故障做出反应。

生命周期监控是通过使用一条能够被监控者接收的:class:`Terminated`消息来实现的，其默认的处理逻辑是抛出一个 :class:`DeathPactException`，如果未以其他方式处理的话。开始监听:class:`Terminated`消息，需要调用``ActorContext.watch(targetActorRef)``.  要停止监听, 则调用
``ActorContext.unwatch(targetActorRef)``. 一个重要的属性是，消息总是会发送，无论监控请求和目标终止所发生的先后顺序是怎样的。也就是，即使注册时目标已经死去，你仍然能够得到这条（终止）消息。

监控特别有用，如果监管者不能够简单重启其子actor，而必须终止的话，例如，在actor初始化出错的情况下。在那种情况下，监管者应当监控哪些子actor，并且重新创建它们，或者安排自己后续重试。

另一个常见的使用场景是，actor在缺少一个外部资源的情况下应当失败，而这一外部资源是子actor之一。如果第三方通过 ``system.stop(child)`` 方法或发送一个
:class:`PoisonPill`消息来终止这一子actor，则监管者也会受到影响。

One-For-One 策略 vs. All-For-One 策略
---------------------------------------------

Akka自带两个监管策略类：:class:`OneForOneStrategy` 和 :class:`AllForOneStrategy`。两者都配置了从异常类型到监管指令(参见
:ref:`above <supervision-directives>`)的映射 ，并且限制了子节点被终止之前所允许的出错频率范围。它们之间的差异是，前者只对出错的子actor应用所得到的指令，而后者还将其应用到所有的邻居actor。正常情况下，你应当使用:class:`OneForOneStrategy`，它也是当没有显式指定策略时的默认值。

 :class:`AllForOneStrategy` 适用的场景是，全体子actor之间具有如此紧密的依赖关系，以至于一个子actor出错会影响其他子actor的功能，也就是，它们之间相互纠缠。因为重启并不会清除邮箱，最好的方案常常是在子actor出错时终止它们，并且从监管者显式地重建它们（通过观察子actor的生命周期）；否则你必须确保，对于其中任一actor，接收一个重启之前入队的消息并且后续处理，是没有问题的。

使用all-for-one策略，正常地停止一个子actor（也就是，并非作为对故障的响应而停止）并不会自动终止其他子actor；但这样的行为能够通过观察其生命周期来很简单地实现：如果:class:`Terminated`消息没有被监管者处理，则它会抛出一个:class:`DeathPactException`，这（取决于监管者）会重启它，并且默认的:meth:`preRestart`动作会终止全部的子actor。当然这也可以显式地被处理。

请注意，从一个all-for-one监管者创建一次性actor，必定导致由这一临时actor所上报的故障影响所有永久actor。如果这不是想要的行为，则安装一个中间监管者；这能够通过为这一辅助actor而声明一个大小为1的路由，而很简单地实现，参见:ref:`routing-scala` 或 :ref:`routing-java`。

