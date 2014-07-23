.. _addressing:

Actor引用，路径和地址
=====================================

本节描述了如何在一个可能为分布式的actor系统中如何识别以及定位一个actor。这关系到了下列中心思想： :ref:`actor-systems` 形成内在的监管层次结构，以及actor之间的通信对于它们在多个网络节点之间的分布是透明的。

.. image:: ActorPath.png

上图展示了actor系统中最重要的实体之间的关系，请继续阅读以了解详情。

Actor引用是什么？
---------------------------

Actor引用是:class:`ActorRef`的子类，它的最重要功能是支持向其所代表的actor发送消息。每个actor可通过 :meth:`self`方法来访问它的标准（本地）引用，在发送给其它actor的消息中也默认包含这个引用。反过来，在消息处理过程中，actor可以通过 :meth:`sender()` 方法来访问当前消息发送者的引用。

取决于actor系统的配置，能够支持几种不同类型的actor引用：

- 纯本地引用被用于配置为不支持网络功能的actor系统中。这些actor引用不能在穿越网络连接而被发送到远程JVM的情况下正常工作。
- 本地actor引用，在远程处理开启的情况下，其所支持的actor系统必须对同一JVM内的actor引用提供网络功能。为了能够在发送到其他网络节点时仍然能够被找到，这些引用包含协议和远程地址信息。
- 本地actor引用有一个子类型用于路由（也就是混入了 :class:`Router` trait的actor）。 它的逻辑结构与之前提到的本地引用是一样的，但是向它们发送的消息却会被直接分发到它的子actor之一。
- 远程actor引用所表示的actor可以通过远程通讯来访问，也就是从别的jvm向他们发送消息时，Akka会透明地对消息进行序列化并发送到远程JVM。
- 有几种特殊的actor引用类型，在实际使用中，其行为类似于本地actor引用：
  - :class:`PromiseActorRef` 表示一个 :meth:`Promise` ，其目的是被一个actor的响应来完成，这类actor引用是由 :meth:`akka.pattern.ask` 创建的。
  -	:class:`DeadLetterActorRef` 是死信服务的默认实现，所有接收方被关闭或不存在的消息都被路由到此处。
  - :class:`EmptyLocalActorRef` 是Akka在查找一个不存在的本地actor路径时所返回的东西：它与 :class:`DeadLetterActorRef` 等价，但是它保留了其路径，因此Akka可以在网络上将其传输，并且可以将其与其路径上其他现有的actor引用进行比较，其中某些引用可能是在该actor死去之前获取的。
- 其次有一次性的内部实现，你应当永远不会真正见到：
  - 有一种actor引用并不表示任何actor，只是作为根守卫actor的伪监管者存在，我们称它为“时空气泡漫步者”。
  -	在用于actor创建的设施之前所启动的第一个日志服务是一个伪actor引用，它接收日志事件并直接显示到标准输出；它就是 :class:`Logging.StandardOutLogger`。

Actor路径是什么?
----------------------

由于actor是以一种严格层次化的形式来创建的，通过递归跟踪父子actor之间的监管链接直到actor系统的根，能够得到一个唯一的actor名称序列。这个序列可以被看作文件系统中的文件夹嵌套，所以我们采用“路径”这一名称来指代它。跟真正的文件系统中一样，这里也存在“符号链接”，也就是，一个actor可通过不止一个路径访问，其中除了一个之外，其它的路径都涉及到一些转化，将路径中的一部分与actor实际的监管祖先路线解耦。这些特点将在下面的小节中介绍。

一个actor路径包含一个标识actor系统的锚点，紧随其后是路径元素的拼接，从根守卫actor到指定的actor；这些路径元素是路径穿过的actor名字，以"/"分隔。

Actor引用和路径之间有什么区别？
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

一个actor引用指明了一个单独的actor，并且引用的生命周期对应了相应actor的生命周期；一个actor路径表示了一个名字，它可能会，也可能不会为一个actor所占据，而且路径自身并不具有生命周期，它永远不会变为无效。你可以创建一个actor路径而不必创建对应的actor，但是你不可能创建一个actor引用而不创建相应的actor。

.. note::

  这一定义对于 ``actorFor`` 并不成立，这是 ``actorFor`` 被废弃并提倡使用 ``actorSelection`` 的原因之一。

你可以创建一个actor，终止它，然后创建一个具有相同actor路径的新actor。新创建的actor是这个actor的一个新的实例。它不是同一个actor。一个指向旧的actor实例的引用对于新的实例是无效的。发往旧actor引用的消息并不会被发送到新的实例，尽管它们具有相同的路径。

Actor路径锚点
^^^^^^^^^^^^^^^^^^

每一条actor路径都有一个地址部分，描述访问相应actor所需的协议和位置，之后是层次结构中从根往上所经过的actor的名字。例如::

  "akka://my-sys/user/service-a/worker1"                   // 纯本地
  "akka.tcp://my-sys@host.example.com:5678/user/service-b" //  远程

这里, akka.tcp 是2.2版本中默认的远程传输协议；其它的传输协议都是可插拔的。一个使用UDP的远程主机可通过使用 ``akka.udp`` 来访问。对主机和端口部分的诠释（上例中，也就是serv.example.com:5678）决定于所使用的传输机制，但它必须遵守URI结构规则。

逻辑Actor路径
^^^^^^^^^^^^^^^^^^^

顺着父监管链一直走到根守卫actor所得到的唯一路径被称为逻辑actor路径。这个路径与actor的创建族谱完全吻合，所以当actor系统的远程远程处理配置（连同路径中的地址部分）设置好后，它就被彻底确定了。

物理Actor路径
^^^^^^^^^^^^^^^^^^^^

逻辑Actor路径描述了一个actor系统内部的功能位置，而基于配置的远程部署意味着一个actor可能在父actor之外的另外一台网络主机上被创建，也就是，在另一个actor系统中。在这种情况下，从根穿过actor路径必定要穿越网络，这是一个昂贵的操作。因此，每一个actor同时还有一条物理路径，从actor对象实际所在的actor系统的根actor开始。在访问actor时使用物理路径作为发送方引用，能够让接收方直接回复到这个actor上，将路由所导致的延迟降到最低限度。

一个重要的方面是，物理路径决不会跨越多个actor系统或虚拟机。这意味着一个actor的逻辑路径（监管层次）和物理路径（actor部署）可能会分叉，如果其祖先之一被远程监管。

如何获得Actor引用？
----------------------------------

大体有两类获取actor引用的方式：通过创建actor或通过查找actor，而后一种功能有两种创建actor引用的风格，通过具体actor路径，以及查询逻辑actor层次结构。

创建Actor
^^^^^^^^^^^^^^^

一个actor系统通常是通过使用 :meth:`ActorSystem.actorOf` 方法在守卫actor之下创建一些actor来启动，然后在创建好的actor之内使用 :meth:`ActorContext.actorOf` 来生成actor树。这些方法返回指向新创建的actor的一个引用。每个actor能够（通过其 ``ActorContext`` ）直接访问其父actor，自己，机器子actor的引用。这些引用可以随消息发往其他actor，允许它们直接回复。

通过具体的路径来查找actor
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

此外，actor引用还可使用 :meth:`ActorSystem.actorSelection` 方法来查找。这个选择可以被用来与所指的actor通信，并且选择所对应的actor在发送每个消息时都会被查找。

要获取一个绑定到特定actor生命周期的 :class:`ActorRef`， 你需要向该actor发送一条消息，如内建的 :class:`Identify` 消息，并且使用此actor的响应之中的 ``sender()`` 引用。

.. note::

  ``actorFor`` 已被废弃，推荐使用 ``actorSelection`` ，因为通过 ``actorFor`` 所获取的引用，对于本地和远程actor具有不同的行为。
  在本地actor的情形下，所指的actor需要在查找之前存在，否则所获取的引用会是一个 :class:`EmptyLocalActorRef`。
  即使具有相同路径的actor在获取actor引用之后被创建，这也成立。
  对于使用 `actorFor` 获取的远程actor引用，其行为是不同的，向这样一个引用发送消息，在后台针对每次消息发送都会根据路径在远程系统中查找actor。

绝对路径 vs. 相对路径
```````````````````````````

除了 :meth:`ActorSystem.actorSelection` 还有一个 :meth:`ActorContext.actorSelection`, 它是任何actor内部都可用的一个 ``context.actorSelection``. 其产生一个actor 选择 的方式跟 :class:`ActorSystem` 上的双胞胎非常类似, 但是它不从actor树根开始查找路径，而从当前actor开始。包含两个点 (``".."``) 的路径元素可被用来访问父actor。例如你可以想一个特定的邻居发送消息::

  context.actorSelection("../brother") ! msg

绝对路径当然也可以在 `context` 上按照通常的方式来查找，也就是

.. code-block:: scala

  context.actorSelection("/user/serviceA") ! msg

其工作的方式符合预期。

查询逻辑Actor树
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

由于actor系统形成了一个类似文件系统的层次结构，对actor路径的匹配方式可能Unix Shell中所支持的方式相同：你可以将路径（中的一部分）替换为通配符(`«*»` and `«?»`)，来形成对0个或多个实际actor的匹配。由于匹配的结果不是一个单独的actor引用，它拥有不同的类型 :class:`ActorSelection` ，这个类型不能支持 :class:`ActorRef` 的整个操作集合。同样，路径选择也可以通过 :meth:`ActorSystem.actorSelection` 和
:meth:`ActorContext.actorSelection` 方法来生成，并且确实支持发送消息::

  context.actorSelection("../*") ! msg

会将msg发送给包括当前actor在内的所有兄弟。对于用 actorFor 获取的actor引用，为了执行消息发送，会对监管层次进行遍历。由于在消息到达其接收者的过程中，与查询条件匹配的actor集合会发生变化，要监视一个选择的死活状态变化是不可能的。如果要做到这一点，通过发送一个请求并收集所有的响应，提取所有的发送方引用，然后监视所有所发现的具体actor来解析不确定性。这种解析actor 选择的方式会在未来的版本中进行改进。

.. _actorOf-vs-actorSelection:

总结: actorOf vs. ``actorSelection`` vs. actorFor
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. note::

  以上小节中所详细描述的内容可以按照如下被概括并轻松地记忆：

  - actorOf 永远都只会创建一个新的actor，这个新的actor被创建为此方法所调用的环境（可以是任何actor或actor系统）的一个直接子actor。

  - ``actorSelection`` 只会在消息被发送时查找已有的actor，也即，在actor选择被创建时，不会创建actor，或验证actor是否存在。

  - actorFor（已弃用，推荐使用actorSelection） 永远都查找一个已存在的actor，即不会创建一个。

Actor引用和路径等价性
---------------------------------
 ``ActorRef`` 的等价性跟 ``ActorRef`` 对应目标actor实例的意图相匹配。两个actor引用在它们具有相同的路径并且指向相同actor实例时，会比较为相等。一个指向已终止的actor的引用跟一个指向相同路径的另一个（重建的）acotr的引用并不会比较为相同。注意一点，由错误所导致的actor重启仍然意味着它是相同的actor实例，也即，重启对于 ``ActorRef`` 的消费者是不可见的。

使用 ``actorFor`` 所获取的远程actor引用并不包含其对应的actor的完整身份信息，因此这样的引用同使用 ``actorOf``, ``sender`` 或r``context.self`` 所获取的引用的比较，结果并不是相等的。因此 ``actorFor`` 已废弃，提倡使用 ``actorSelection`` 。

如果你需要在集合中跟踪actor引用，并且不关心准确的actor实例，你可以使用 ``ActorPath`` 作为key，因为当比较actor路径时，目标actor的身份信息并不被考虑。

重用actor路径
-------------------

当actor被终止时，其引用将指向死信信箱，DeathWatch会发布其最终的状态转换，并且通常它并不被预期能够重新活过来（因为actor生命周期不允许这样的情况出现）。
尽管有可能在稍后的时间创建一个具有相同路径的actor-仅仅由于如果不保持一个集合来存放曾经创建的所有actor就不可能保证对立面（simply due to it being impossible to enforce the opposite without keeping the set of all actors ever created available）-这并不是好的实践: 使用 ``actorFor`` 所获取的远程actor引用在“死去”之后突然又可以工作了，但是这一状态转换与其他事件之间的顺序没有任何保证，因此这个路径对应的新actor可能会接收到目标为前一个actor的消息。

在非常特殊的情况下，这可能是正确的做法，但是要确保将精确处理这件事的职责限定到actor的监管者，因为那是唯一能够可靠地检测此actor名称正确注销的actor，在此前创建新的子actor会失败。

在测试过程中也可能需要这样做，当测试主体依赖于在特定路径上进行初始化。在那种情形下，最好mock其监管者，从而使得它将Terminated消息转发到测试过程中正确的节点，允许此节点等待这一actor名称的正确注销。

与远程部署的交互
------------------------------------

当一个actor创建一个子actor时，actor系统的部署者会决定将这个新的actor部署在同一个JVM还是其它节点中。在后一种情况下，此actor的创建将通过网络连接来触发于另一个JVM上，因此也是在另一个actor系统中。远程系统会将新的actor放置于专用为此预留的一个特殊路径下，并且新actor的监管者将是一个远程actor引用（表示触发其创建的actor）。在这种情况下，:meth:`context.parent` (监管者引用) 和 :meth:`context.path.parent` (actor路径中的父节点)并不表示相同的actor。然而，在监管者之中查找子actor的名称将在远程节点上找到它，保留逻辑结构。例如当向一个未加分析的actor引用发送消息时。

.. image:: RemoteDeployment.png

路径中的地址部分用来做什么？
----------------------------------

在网络上发送一个actor引用时，actor是它的路径来表示的。因此，它的路径必须对向其所代表的actor发送消息所需的所有信息进行完整的编码。这一点是通过在路径字符串的地址部分中编入协议、主机和端口来实现的。当actor系统从一个远程节点接收到一个actor路径时，它会检查那个路径的地址部分是否与这个actor系统的地址匹配，如果匹配，那么会此路径解析为本地引用，否则解析为一个远程actor引用。

.. _toplevel-paths:

Actor路径的顶层域(Scope)
--------------------------------

路径层次结构的根部是根守卫actor，所有其他的actor都可以在它上面找到；其名称为 ``"/"``。下一个层次上由以下actor组成：

``"/user"`` 是所有由用户创建的顶层actor的监管者，用 :meth:`ActorSystem.actorOf` 创建的actor位于此actor下。
``"/system"`` 是所有由系统创建的顶层actor，如日志监听器，或者根据配置在actor系统启动时自动部署的actor，的监管者。
``"/deadLetters"`` 是死信actor，所有发往已停止或不存在的actor的消息会被改道发送到这里。
``"/temp"`` 是所有系统创建的生命周期短暂actor(也就是那些用于 :meth:`ActorRef.ask` 的实现中的actor)的监管者。
``"/remote"`` 是一个假造的路径，其下用来存放监管者为远程actor引用的所有actor。
类似这样的构建actor名字空间的需求，来源于一个核心的非常简单的设计目标：层次结构中的一切都是actor，并且所有的actor以相同的方式来运转。因此你不仅能够查找你所创建的actor，还可以查找系统守卫actor，并且向其发送消息（在这种情况下，消息会被负责任地丢掉）。这一强有力的原则意味着没有什么诡异之处需要记住，它使得整个系统更加统一和一致。

付过你需要阅读更多关于actor系统顶层结构的内容，请参阅 :ref:`toplevel-supervisors`.

