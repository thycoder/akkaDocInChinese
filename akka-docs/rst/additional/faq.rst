常见问题(FAQ)
==========================

Akka 项目
^^^^^^^^^^^^

Akka这一名字是怎么来的？
-----------------------------------

这个名字来自于一座风景优美的瑞典 `山峰 <https://lh4.googleusercontent.com/-z28mTALX90E/UCOsd249TdI/AAAAAAAAAB0/zGyNNZla-zY/w442-h331/akka-beautiful-panorama.jpg>`_
，它矗立在瑞典北部名为Laponia的地方。这座山峰有时也被称作 'The Queen of Laponia'.

Akka 还是一个女神的名字，来自于Sámi (瑞典原住民)神话。她是代表世间所有美好事物的神。这座山可以被看做这个女神的象征。

另外，名称 AKKA是Actor Kernel中字母A和K的回文。

Akka 还是:

* 瑞典作家Selma Lagerlöf的 `The Wonderful Adventures of Nils <http://en.wikipedia.org/wiki/The_Wonderful_Adventures_of_Nils>`_ 中 Nils 穿越瑞典时所骑的鹅的名字。
* 表示 '讨厌的老女人' 的芬兰语以及 '大姐' 在印度语言 Tamil, Telugu, Kannada 和 Marathi 中的表示。
* 一个 `字体 <http://www.dafont.com/akka.font>`_
* 一个摩洛哥小镇
* 一个接近地球的小行星

Actors 一般知识
^^^^^^^^^^^^^^^^^

当我在Actor中使用Future时，sender()/getSender() 会消失，为什么?
-------------------------------------------------------------------

当使用future回调时，在actor内部你需要仔细地避免捕获外围actor的引用，也就是，不要从回调方法内部调用外围actor的方法或访问其可变状态。这会打破actor的封装，并且导致同步bug以及竞态条件，因为回调将同外围的actor并发地被调度。遗憾的是在编译时还没有办法可以检测这些非法访问。

更多信息请参阅 :ref:`jmm-shared-state`.

为何会遇到 OutOfMemoryError?
---------------------

OutOfMemoryError的可能原因很多。 录入，在一个纯粹基于push的系统中，消息消费者可能比相应的消息生产者更慢，你必须添加某种类型的消息流量控制。否则消息将被排入消费者信箱的队列，从而占满堆内存。

一些用于激发灵感的文章:

* `Balancing Workload across Nodes with Akka 2 <http://letitcrash.com/post/29044669086/balancing-workload-across-nodes-with-akka-2>`_.
* `Work Pulling Pattern to prevent mailbox overflow, throttle and distribute work <http://www.michaelpollmeier.com/akka-work-pulling-pattern/>`_

Actors Scala API
^^^^^^^^^^^^^^^^

我如何能得到报告 `receive` 中缺少消息的编译时错误？
--------------------------------------------------------------------

一种可帮助你得到没有处理一条应当处理的消息的编译时警告的解决方式是，定义你的actor输入与输出消息时，让它们实现基本的trait，然后执行一个匹配，它的完备性将被检查。

这里是一个例子，在其中编译器将会警告recive中的匹配不完备:

.. includecode:: code/docs/faq/Faq.scala#exhaustiveness-check

远程处理
^^^^^^^^

我想要向一个远程系统发送消息，但是让它什么都不做。
-------------------------------------------------------------

确保你在客户端和服务端都启用了远程处理。两者都需要配置主机和端口，并且你将需要知道服务器端口;客户端在大多数情况下可以使用自动端口（也就是将端口配置为0）。如果两个系统都运行在同一个网络主机上，它们的端口必须是不同的。
如果你仍然啥都看不到，查看远程生命周期事件日志告诉你的信息（通常日志级别为INFO），或者打开 :ref:`logging-remote-java` 来查看所有收发的消息(日志级别为DEBUG)。

当调试远程问题时，我应当启用哪些选项？
------------------------------------------------------------

查看 :ref:`remote-configuration-java`, 典型的候选者为:

* `akka.remote.log-sent-messages`
* `akka.remote.log-received-messages`
* `akka.remote.log-remote-lifecycle-events` (这还包括反序列化错误)

远程actor的名字是什么？
-----------------------------------

当你要向一个远程主机上的actor发送消息时，你需要知道它的 :ref:`完整路径 <addressing>`, 形式如下 ::

    akka.protocol://system@host:1234/user/my/actor/hierarchy/path

注意这里你所需的所有部分:

* ``protocol`` 是用于跟远程系统通信的协议。大多数情况下是 `tcp`.

* ``system`` 是远程actor系统的名称 必须精确匹配，大小写敏感!)

* ``host`` 是远程系统的 IP 地址或 DNS 名, 并且它必须匹配那个系统的配置 (也就是 `akka.remote.netty.hostname`)

* ``1234`` 是远程系统监听连接和接收消息所哟过的端口号

* ``/user/my/actor/hierarchy/path`` 是远程actor在远程系统的监管层次结构中的的绝对路径, 包括系统的守卫actor(也就是 ``/user``, 还有其他的actor如持有日志管理器的 ``/system`` , 持有同 `ask()` 使用的临时actor引用的 ``/temp`` , 启用远程部署的 `/remote` ，等等。); 这匹配了actor在远程主机上打印其 ``self``引用的方式，例如在日志输出中。

来自一个远程actor的回复为何未收到?
-------------------------------------------------

最常见的原因是，本地系统的名称(也就是上述回答中的
``system@host:1234`` 部分)从远程网络位置不可达，例如由于  ``host`` 被配置为 ``0.0.0.0``,
``localhost`` 活着一个NAT 的 IP 地址。

消息投递有多么可靠?
-------------------------------------

一般规则是 **at-most-once delivery**, 也即是，没有投递保证。更强的可靠性可以基于此而构建，并且Akka提供了工具。

在 :ref:`message-delivery-reliability` 中阅读更多内容。

调试
^^^^^^^^^

如何打开debug日志？
-------------------------------

要在你的actor系统中打开debug日志，请在你的配置中添加下列内容 ::

    akka.loglevel = DEBUG  

要启用不同类型的debug日志，请在你的配置中国添加以下内容 :

* ``akka.actor.debug.receive`` 将记录所有发往actor的消息，如果那个actor的 `receive` 方法是一个 ``LoggingReceive``

* ``akka.actor.debug.autoreceive`` 将记录发往所有actor的所有 *特殊* 消息，类似于 ``Kill``, ``PoisonPill`` 等等。

* ``akka.actor.debug.lifecycle`` 会记录所有actor的所有生命周期事件

更多内容请参阅文档中的 :ref:`logging-java` 和 :ref:`actor.logging-scala`.
