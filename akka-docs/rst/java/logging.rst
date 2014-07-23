.. _logging-java:

################
 日志
################

Akka中的日志并没有依赖一个特定的日志后端。默认情况下，日志消息被打印到STDOUT, 但你可以插入一个SLF4J 日志处理器，或者你自己的日志处理器。日志是异步执行的，从而确保日志对性能的冲击最小化。日志通常意味着IO和加锁，如果同步执行的话，会减慢你的代码中的业务操作。 

如何使用日志
==========

创建一个 ``LoggingAdapter`` 并使用 ``error``, ``warning``, ``info``, 或 ``debug`` 方法,
如这个例子所阐述:

.. includecode:: code/docs/event/LoggingDocTest.java
   :include: imports

.. includecode:: code/docs/event/LoggingDocTest.java
   :include: my-actor


``Logging.getLogger`` 的第一个参数还可以是任何  :class:`LoggingBus` ，特别是 ``system.eventStream()`` ;在所描述的例子中，actor系统的地址被包含在日志源的 ``akkaSource`` 表示形式中。(参见 `MDC中的日志线程和Akka源`_) 然而在第二个例子中，这不是自动完成的。 ``Logging.getLogger`` 的第二个参数是日志通道的源。这个源对象被翻译为一个String，所遵循的规则如下:

  * 如果它是一个 Actor 或 ActorRef, 则使用其路径
  * 如果是String，则直接使用
  * 如果是class 则使用其 simpleName 来近似表示
  * 在所有其他情况下，使用其类的simpleName

日志消息可包含参数占位符 ``{}``, 如果这一日志级别被启用，则占位符会被替换。给出的参数多于占位符，会导致日志后边被添加一个警告(也就是，在同一行，以相同的严重级别)。你可以将一个Java数组作为唯一的替换参数传入，则它的元素会被单独处理:

.. includecode:: code/docs/event/LoggingDocTest.java#array

日志源的Java :class:`Class` 也被包含在生成的 :class:`LogEvent` 中。 在简单字符串的情形中，这会被替换为一个 “标记”
类 :class:`akka.event.DummyClassForStringSources` ，从而允许这种情况被特殊对待，例如，在SLF4J事件监听器中，它会使用字符串而不是这个类名来查找所要使用的日志处理器实例。

记录死信日志
-----------------------

默认情形下，发给死信的日志会以info级别被记录。死信的存在并不一定表示有问题，但是它可能会，因此它们默认会被记录日志。在多条消息之后，这个日志功能会被关闭，避免淹没其他日志。你可以彻底禁用这种日志，或者调整所记录的死信数量。在系统关闭是，有可能你会看到死信，因为actor信箱中待处理的消息会被发送到死信。你还可以在关闭过程中禁用死信日志。

.. code-block:: ruby

    akka {
      log-dead-letters = 10
      log-dead-letters-during-shutdown = on
    }

要进一步定制这一日志，或针对死信采取其他行动，你可以订阅 :ref:`event-stream-java`.

日志的辅助选项
-------------------------

Akka有一些针对特别底层的调试的配置选项，这对于开发者而不是运维者有意义。

你几乎肯定需要将日志设置为DEBUG来使用下面选项:

.. code-block:: ruby

    akka {
      loglevel = "DEBUG"
    }

这个配置选项在你想知道Akka加载了什么配置时非常好用:

.. code-block:: ruby

    akka {
      # Log the complete configuration at INFO level when the actor system is started.
      # This is useful when you are uncertain of what configuration is used.
      log-config-on-start = on
    }

如果你想要所有自动接收的消息的非常详细的日志:

.. code-block:: ruby

    akka {
      actor {
        debug {
          # enable DEBUG logging of all AutoReceiveMessages (Kill, PoisonPill et.c.)
          autoreceive = on
        }
      }
    }

如果你想要Actor所有生命周期变化的非常详细的日志(重启，死亡，等等):

.. code-block:: ruby

    akka {
      actor {
        debug {
          # enable DEBUG logging of actor lifecycle changes
          lifecycle = on
        }
      }
    }

如果你想要继承LoggingFSM的FSM Actor的所有时间，生命周期变化和定时器的非常详细的日志:
If you want very detailed logging of all events, transitions and timers of FSM Actors that extend LoggingFSM:

.. code-block:: ruby

    akka {
      actor {
        debug {
          # enable DEBUG logging of all LoggingFSMs for events, transitions and timers
          fsm = on
        }
      }
    }
	
如果你想要监控Actor.eventStream的订阅(订阅/取消订阅):

.. code-block:: ruby

    akka {
      actor {
        debug {
          # enable DEBUG logging of subscription changes on the eventStream
          event-stream = on
        }
      }
    }

.. _logging-remote-java:

远程日志辅助选项
--------------------------------

如果你要在DEBUG日志级别看到所有远程发送的消息:
(这是在它们被发送到传输层时，而不是被Actor发送时记录的)

.. code-block:: ruby

    akka {
      remote {
        # If this is "on", Akka will log all outbound messages at DEBUG level,
        # if off then they are not logged
        log-sent-messages = on
      }
    }


如果你要在DEBUG日志级别看到所有远程接收的消息:
(这是在它们被接收到传输层时，而不是被Actor接收时记录的)

.. code-block:: ruby

    akka {
      remote {
        # If this is "on", Akka will log all inbound messages at DEBUG level,
        # if off then they are not logged
        log-received-messages = on
      }
    }


如果你要在INFO日志级别看到所有载荷尺寸的大于指定字符数限制的消息:

.. code-block:: ruby

    akka {
      remote {
        # Logging of message types with payload size in bytes larger than
        # this value. Maximum detected size per message type is logged once,
        # with an increase threshold of 10%.
        # By default this feature is turned off. Activate it by setting the property to
        # a value in bytes, such as 1000b. Note that for all messages larger than this
        # limit there will be extra performance and scalability cost.
        log-frame-size-exceeding = 1000b
      }
    }

还请查看 TestKit的日志选项: :ref:`actor.logging-java`.

关闭日志
----------------

要关闭日志，你可以将日志级别设置为 ``OFF`` 。

.. code-block:: ruby

  akka {
    stdout-loglevel = "OFF"
    loglevel = "OFF"
  }

 ``stdout-loglevel`` 仅会在系统启动或关闭的过程中起作用，将其也设置为 ``OFF`` 可确保系统启动或停止过程中不记录任何日志。

日志处理器
=======

日志是通过事件总线异步记录的。日志事件是由一个日志处理器actor来处理的，它会按照日志事件发送的顺序来接收它们。

你可以配置系统启动时创建何种事件处理器用于监听日志事件。这可以使用 :ref:`configuration` 中的 ``loggers`` 来配置。这里你还可以定义日志级别。

.. code-block:: ruby

  akka {
    # Loggers to register at boot time (akka.event.Logging$DefaultLogger logs
    # to STDOUT)
    loggers = ["akka.event.Logging$DefaultLogger"]
    # Options: OFF, ERROR, WARNING, INFO, DEBUG
    loglevel = "DEBUG"
  }

默认的日志处理器将日志写入STDOUT，并且默认会被注册。它并不用于生产环境。还可以使用 'akka-slf4j' 模块中的 :ref:`slf4j-java` 处理器。

创建监听器的例子:

.. includecode:: code/docs/event/LoggingDocTest.java
   :include: imports,imports-listener

.. includecode:: code/docs/event/LoggingDocTest.java
   :include: my-event-listener

.. _slf4j-java:

启动和关闭期间向stdout打印日志
=============================================

在actor系统启动和关闭期间，所配置的 ``loggers`` 不会被使用。相反，日志消息会被打印到 stdout (System.out). 这个stdout日志处理器的默认日志级别是 ``WARNING`` 并且它可以通过设置 ``akka.stdout-loglevel=OFF`` 而被彻底关闭。

SLF4J
=====

Akka 为 `SL4FJ <http://www.slf4j.org/>`_ 提供了一个日志处理器。这个模块在 'akka-slf4j.jar' 中。它只有一个依赖; slf4j-api jar. 运行时你还需要一个 SLF4J 后端, 我们推荐 `Logback <http://logback.qos.ch/>`_:

  .. code-block:: xml

     <dependency>
       <groupId>ch.qos.logback</groupId>
       <artifactId>logback-classic</artifactId>
       <version>1.0.13</version>
     </dependency>

你需要在 :ref:`configuration` 的 'loggers' 元素中启用 Slf4jLogger. 这里你还需要定义事件总线的日志级别。 更加细粒度的日志级别可以在SLF4J 后端的配置中指定。(例如 logback.xml).

.. code-block:: ruby

  akka {
    loggers = ["akka.event.slf4j.Slf4jLogger"]
    loglevel = "DEBUG"
  }
  
一个陷阱是时间戳是在事件处理器中设置的，而不是在实际记录日志时。

每个日志事件所选用的SLF4J logger的选择，是基于创建 :class:`LoggingAdapter` 所指定的事件源的 :class:`Class` , 除非它作为一个字符串而直接给定，在这种情况下这个字符串会被使用(也就是 ``LoggerFactory.getLogger(Class c)`` 在第一种情况下被使用，而 ``LoggerFactory.getLogger(String s)`` 在第二种情况下被使用).

.. note::
  注意actor系统的名字会被附加到日志源 :class:`String` ，如果 LoggingAdapter 创建时向工厂传递了一个 :class:`ActorSystem` 。如果意图不是如此，则传入一个 :class:`LoggingBus` 作为替代，如下所示:

.. code-block:: scala

  final LoggingAdapter log = Logging.getLogger(system.eventStream(), "my.string");

MDC中的日志线程和Akka源
-------------------------------------

因为日志是异步记录的，执行日志的线程会被捕获到映射诊断上下文 (MDC) 中，属性名称为 ``sourceThread`` 。使用Logback时，线程名字可以在模式布局配置器中通过 ``%X{sourceThread}`` 指示符来获取 ::

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%date{ISO8601} %-5level %logger{36} %X{sourceThread} - %msg%n</pattern>
    </encoder>
  </appender>

.. note::
  
  一个可能的好主意是，在应用程序中非Akka的部分也使用 ``sourceThread`` MDC 值，从而让这个属性在日志中一致地可见。

另一个有用的设施是，Akka在actor中实例化一个日志处理器时，会捕获actor的地址，这意味着完整的实例标识对相关的日志消息可用，例如，路由器 的成员。这个信息在MDC中的属性名是 ``akkaSource`` ::

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%date{ISO8601} %-5level %logger{36} %X{akkaSource} - %msg%n</pattern>
    </encoder>
  </appender>

更多关于这个属性包含什么-也针对非actor代码-请查看 
`如何使用日志`_.

MDC中日志输出的更精确的时间戳
------------------------------------------------

Akka的日志是异步的，这意味着log条目的时间戳是底层日志实现代码被调用的时间，这乍一看很奇怪。如果你想更精确地输出时间戳，使用MDC属性 ``akkaTimestamp``::

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%X{akkaTimestamp} %-5level %logger{36} %X{akkaSource} - %msg%n</pattern>
    </encoder>
  </appender>


应用程序定义的MDC值
-------------------------------------

Slf4j中的一个有用特性是 `MDC <http://logback.qos.ch/manual/mdc.html>`_,
Akka 有一种方法可以让应用程序指定自定义值，你仅需获取特定的 :class:`LoggingAdapter`,  :class:`DiagnosticLoggingAdapter` 。为了获取它，你将使用使用接收一个UntypedActor作为日志源的工厂方法:

.. code-block:: scala

    // Within your UntypedActor
    final DiagnosticLoggingAdapter log = Logging.getLogger(this);

你一旦获取这个日志处理器，你只需在记录日志前添加自定义值。这样，这些值将刚好在输出这条日志之前被放入 SLF4J MDC 并且随后被删除。

.. note::
  
  清理(删除)应当由actor在最后完成，否则，下一条消息将记录相同的mdc值，如果它没有被设置为一个新的map。使用 ``log.clearMDC()``.

.. includecode:: code/docs/event/LoggingDocTest.java
    :include: imports-mdc

.. includecode:: code/docs/event/LoggingDocTest.java
    :include: mdc-actor

现在，这些值将在MDC中可用，因此你可以再布局模式中使用它们 ::

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>
        %-5level %logger{36} [req: %X{requestId}, visitor: %X{visitorId}] - %msg%n
      </pattern>
    </encoder>
  </appender>

