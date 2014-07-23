.. _configuration:

配置
=============

你可以在不定义任何配置的情况下开始使用Akka，因为合理的默认值已经被提供。后续你可能需要修改这些配置，从而修改默认行为，或适配特定的运行时环境。你可能修改的配置的典型示例有：

* 日志级别和日志记录器后端
* 启用远程处理
* 消息序列化器
* 定义路由
* 调整调度器(dispatcher)

Akka 使用 `Typesafe Config Library
<https://github.com/typesafehub/config>`_, 对于你自己的应用程序，或者无论是否使用Akka所构建的库，它也是进行配置的一个很好的选项。这个库使用Java实现，没有任何外部依赖；你应当查看其文档（特别是关于 `ConfigFactory
<http://typesafehub.github.io/config/v1.2.0/com/typesafe/config/ConfigFactory.html>`_),
以下只是概要.

.. warning::

   如果你通过Scala REPL 2.9.x版本来使用Akka，并且你不为ActorSystem提供自己的ClassLoader，使用"-Yrepl-sync" 启动REPL来绕过REPL所提供的Context ClassLoader的一个缺陷。


配置从何处读取？
--------------------------------

Akka的所有配置信息都位于 :class:`ActorSystem` 的实例中, 或者换个说法, 从外界看来, :class:`ActorSystem` 是配置信息的唯一消费者. 在构造一个actor系统时，你可以传入一个 :class:`Config` 对象，如果不传则相当于传入 ``ConfigFactory.load()`` (使用正确的classloader). 这大体上意味着，默认将解析classpath根目录下的所有 ``application.conf``,
``application.json`` 和 ``application.properties``  这些文件—请参阅之前推荐的文档以了解详情. 然后actor系统会斌合并classpath根目录下所有的 ``reference.conf``  来作为其内部使用的备用配置。

.. code-block:: scala

  appConfig.withFallback(ConfigFactory.defaultReference(classLoader))

其中的哲学是代码绝不包含默认值，而是却依赖于它们在随库提供的 ``reference.conf`` 中的值.

系统属性中覆盖的配置具有最高优先级，见 `the
HOCON specification
<https://github.com/typesafehub/config/blob/master/HOCON.md>`_ (靠近末尾的位置). 值得注意的是应用程序配置—默认为 ``application`` —可以使用  ``config.resource`` 属性来覆盖 (更多细节参阅 `Config docs
<https://github.com/typesafehub/config/blob/master/README.md>`_ )。

.. note::

  如果你编写的是一个Akka应用，把配置放在classpath根目录下的 ``application.conf`` 中. 如果你编写的是一个基于Akka的库，把配置放在jar包根目录下的 ``reference.conf`` 中.

当使用 JarJar, OneJar, Assembly 或 any jar-打包器时
------------------------------------------------------

.. warning::
    Akka的配置方式严重依赖于每个模块/jar都有自己的reference。conf文件这一观念，这些文件全都会被配置所发现并加载。不幸的是，这也意味着你如果将多个jar包放入或并入同样的一个jar包中，你还需要合并所有的reference.conf。否则所有默认值将会丢失，并且Akka将不能工作。

如果你使用Maven来打包你的应用，你还可以利用 `Apache Maven Shade Plugin
<http://maven.apache.org/plugins/maven-shade-plugin>`_ 对 `Resource
Transformers
<http://maven.apache.org/plugins/maven-shade-plugin/examples/resource-transformers.html#AppendingTransformer>`_ 的支持，将构建classpath上所有的reference.conf文件合并为一个。 

这个插件的配置看上去如下所示::

    <plugin>
     <groupId>org.apache.maven.plugins</groupId>
     <artifactId>maven-shade-plugin</artifactId>
     <version>1.5</version>
     <executions>
      <execution>
       <phase>package</phase>
       <goals>
        <goal>shade</goal>
       </goals>
       <configuration>
        <shadedArtifactAttached>true</shadedArtifactAttached>
        <shadedClassifierName>allinone</shadedClassifierName>
        <artifactSet>
         <includes>
          <include>*:*</include>
         </includes>
        </artifactSet>
        <transformers>
          <transformer
           implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
           <resource>reference.conf</resource>
          </transformer>
          <transformer
           implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
           <manifestEntries>
            <Main-Class>akka.Main</Main-Class>
           </manifestEntries>
          </transformer>
        </transformers>
       </configuration>
      </execution>
     </executions>
    </plugin>

自定义 application.conf
-----------------------

一个自定义的 ``application.conf`` 可能看起来如下所示::

  # In this file you can override any option defined in the reference files.
  # Copy in parts of the reference files and modify as you please.

  akka {

    # Loggers to register at boot time (akka.event.Logging$DefaultLogger logs
    # to STDOUT)
    loggers = ["akka.event.slf4j.Slf4jLogger"]

    # Log level used by the configured loggers (see "loggers") as soon
    # as they have been started; before that, see "stdout-loglevel"
    # Options: OFF, ERROR, WARNING, INFO, DEBUG
    loglevel = "DEBUG"

    # Log level for the very basic logger activated during ActorSystem startup.
    # This logger prints the log messages to stdout (System.out).
    # Options: OFF, ERROR, WARNING, INFO, DEBUG
    stdout-loglevel = "DEBUG"

    actor {
      provider = "akka.cluster.ClusterActorRefProvider"
      
      default-dispatcher {
        # Throughput for default Dispatcher, set to 1 for as fair as possible
        throughput = 10
      }
    }

    remote {
      # The port clients should connect to. Default is 2552.
      netty.tcp.port = 4711
    }
  }


包含文件
---------------

有时包含另一个配置文件会很有用，例如如果你拥有一个包含所有独立于环境的设置项的 ``application.conf`` ，然后针对特定的环境而覆盖某些设置项。

将系统属性指定为 ``-Dconfig.resource=/dev.conf`` 将加载 ``dev.conf`` file, 而它包含了 ``application.conf``

dev.conf:

::

  include "application"

  akka {
    loglevel = "DEBUG"
  }
更加先进的包含和替换机制在 `HOCON <https://github.com/typesafehub/config/blob/master/HOCON.md>`_ 规范中进行了解释。


.. _-Dakka.log-config-on-start:

为配置记录日志
------------------------

如果系统属性或配置属性 ``akka.log-config-on-start`` 被设置为 ``on``, 那么当actor系统启动时会以INFO级别记录完整的配置。这在你不确定使用了什么配置时，是非常有用的。

如果有疑问，你还可以在使用配置对象构造一个actor系统之前和之后，简单而精细地检查它。

.. parsed-literal::

  Welcome to Scala version @scalaVersion@ (Java HotSpot(TM) 64-Bit Server VM, Java 1.6.0_27).
  Type in expressions to have them evaluated.
  Type :help for more information.

  scala> import com.typesafe.config._
  import com.typesafe.config._

  scala> ConfigFactory.parseString("a.b=12")
  res0: com.typesafe.config.Config = Config(SimpleConfigObject({"a" : {"b" : 12}}))

  scala> res0.root.render
  res1: java.lang.String =
  {
      # String: 1
      "a" : {
          # String: 1
          "b" : 12
      }
  }

每项之前的注释给出了关于此配置项来源的详细信息（文件和行号），以及可能出现的注释，例如，在参考配置中，并入参考配置并由actor系统解析的配置可以使用如下代码来显示:

.. code-block:: java

  final ActorSystem system = ActorSystem.create();
  System.out.println(system.settings());
  // this is a shortcut for system.settings().config().root().render()

关于类加载器（ClassLoader）
-------------------------

在配置文件的某些地方可以指定要被Akka实例化的对象的全类名。这是通过Java反射来实现的，会用到 :class:`ClassLoader` 。 在应用容器或OSGi bundle等有挑战的环境里使用正确的一个，并不总是件容易的事，目前Akka采取的方式是每个 :class:`ActorSystem` 实现存有当前线程的上下文类加载器 (如果可用的话，否则仅使用 ``this.getClass.getClassLoader`` 中自身的类加载器) 并用它来进行所有的反射操作。 这意味着若将Akka放到boot class path中，会在一些奇怪的地方造成 :class:`NullPointerException` : 这个是不支持的。

应用程序专有设置项
-----------------------------
配置还可以被用于应用程序专有设置。一种最佳实践是将这些设置项放到一个 Extension 中,如下所述:

 * Scala API: :ref:`extending-akka-scala.settings`
 * Java API: :ref:`extending-akka-java.settings`

配置多个ActorSystem
--------------------------------

如果你拥有多个 ``ActorSystem`` (或者你正在编写一个库，并且拥有一个与应用程序的ActorSystem分开的另一个ActorSystem) 你可能想隔离每个actor系统的配置。

已知 ``ConfigFactory.load()`` 合并整个class path上具有匹配的名称的所有资源，最简单的方法是利用这一功能，并且在配置文件的层次结构中区分不同的actor系统 ::

  myapp1 {
    akka.loglevel = "WARNING"
    my.own.setting = 43
  }
  myapp2 {
    akka.loglevel = "ERROR"
    app2.setting = "appname"
  }
  my.own.setting = 42
  my.other.setting = "hello"

.. code-block:: scala

  val config = ConfigFactory.load()
  val app1 = ActorSystem("MyApp1", config.getConfig("myapp1").withFallback(config))
  val app2 = ActorSystem("MyApp2",
    config.getConfig("myapp2").withOnlyPath("akka").withFallback(config))

这两个示例阐述了"举起子树"技巧的不同变种：在第一个例子中，从actor系统内部可以访问的配置如下：

.. code-block:: ruby

  akka.loglevel = "WARNING"
  my.own.setting = 43
  my.other.setting = "hello"
  // plus myapp1 and myapp2 subtrees

而在第二个例子中, 仅有“akka”子树被举起, 结果如下：

.. code-block:: ruby

  akka.loglevel = "ERROR"
  my.own.setting = 42
  my.other.setting = "hello"
  // plus myapp1 and myapp2 subtrees

.. note::

   这一配置库真的特别强大，解释其所有特性超出了这里能够接受的范围。特别地，没有涵盖的内容包括如何在其他文件中包括另一个配置文件（在 `包含文件`_ 中能看到一个小例子) 并且通过路径替换的方式来复制配置树之中的一些部分。

你还能在实例化 ``ActorSystem`` 时通过其他方式编程指定并解析配置。

.. includecode:: code/docs/config/ConfigDocSpec.scala
   :include: imports,custom-config

从一个自定义的位置读取配置
--------------------------------------------

你可以在代码中，或使用系统变量来替换或补充功能 ``application.conf`` 。

如果你使用 ``ConfigFactory.load()`` (Akka 的默认行为) ，你可以通过定义
``-Dconfig.resource=whatever``, ``-Dconfig.file=whatever``, 或
``-Dconfig.url=whatever`` 来替换 ``application.conf`` 。

在你使用 ``-Dconfig.resource`` 及其伙伴参数所指定的替换文件中，你可以 ``include "application"`` ，如果你仍然还想使用 ``application.{conf,json,properties}`` 。  在 ``include "application"`` 之前所指定的设置项， 会被所包含的文件所覆盖, 而之后的配置项会覆盖包含的文件。

在代码中，有很多自定义选项。

 ``ConfigFactory.load()`` 有一些重载方法; 这些方法允许你指定一些夹在系统属性 (覆盖现有值) 和默认值 (来自 ``reference.conf``) 之间的东西, 替换常见的 ``application.{conf,json,properties}`` 并代替 ``-Dconfig.file`` 及其伙伴。


``ConfigFactory.load()`` 最简单的变化形式需要传入一个资源基本名myname（代替 ``application`` ); ``myname.conf``,
``myname.json``, 和 ``myname.properties`` 然后就会代替 ``application.{conf,json,properties}`` 而被使用.

最灵活的变化形式需要传入一个 ``Config`` 对象, 你可以使用 ``ConfigFactory`` 中的任一种方法来获取他。 例如，你可以使用
``ConfigFactory.parseString()`` 在代码中放入一个配置字符串，或者你也可以构造一个map并调用 ``ConfigFactory.parseMap()``, 或者你也可以读取一个文件。

你还可以将你的自定义配置与通常的配置相结合，代码如下：

.. includecode:: code/docs/config/ConfigDoc.java
   :include: java-custom-config
   
当操作 ``Config`` 对象时, 牢记这个“蛋糕”有三层:
 
 - ``ConfigFactory.defaultOverrides()`` (系统属性)
 - 应用程序设置
 - ``ConfigFactory.defaultReference()`` (reference.conf)

标准的目标是自定义中间层，而保持另外两层不变。

 - ``ConfigFactory.load()`` 读取整个配置堆栈
 - ``ConfigFactory.load()`` 的重载方法允许你指定一个不同的中间层
 - ``ConfigFactory.parse()`` 的变化形式加载单独的文件或资源

要堆叠两层, 请使用 ``override.withFallback(fallback)``; 尽量将系统属性 (``defaultOverrides()``) 保留在顶层，将 ``reference.conf`` (``defaultReference()``) 保留在底层.

一定要牢记，你经常可以在 ``application.conf`` 中添加另一条 ``include`` 语句而无需编写代码。

``application.conf`` 顶部的include文件会被 ``application.conf`` 的剩余部分覆盖，而底层的include文件会覆盖前边的内容。

Actor 部署配置
------------------------------

针对指定actor的部署设置项可在配置中的 ``akka.actor.deployment`` 部分进行定义。在部署部分，可以定义类似调度器，信箱，路由设置，远程部署等东西。这些特性的配置在详细讲述相应话题的章节中进行了介绍。一个例子可能看起来是这样的：

.. includecode:: code/docs/config/ConfigDocSpec.scala#deployment-section

特定actor的部署部分由actor相对于 ``/user`` 的路径来标识。

你可以使用*作为通配符，匹配actor路径段，因此你可以指定： ``/*/sampleActor`` ，那将匹配层次结构中那一级上所有的 ``sampleActor`` 。你还可以在最后位置使用通配符，匹配特定层级的所有actor： ``/someParent/*`` 。 不含通配符的匹配规则总是比使用通配符的匹配规则优先级更高，因此: ``/foo/bar`` 被认为比 ``/foo/*`` **更具体** ，并且只有最高优先级的匹配规则会被使用。 请注意，它 **不能** 被用于部分匹配路径段, 如这样的: ``/foo*/bar``, ``/f*o/bar`` 等等。

参考配置清单
--------------------------------------

每个Akka模块都有一个参考配置文件，包含默认值。

.. _config-akka-actor:

akka-actor
~~~~~~~~~~

.. literalinclude:: ../../../akka-actor/src/main/resources/reference.conf
   :language: none

.. _config-akka-agent:

akka-agent
~~~~~~~~~~

.. literalinclude:: ../../../akka-agent/src/main/resources/reference.conf
   :language: none

.. _config-akka-camel:

akka-camel
~~~~~~~~~~

.. literalinclude:: ../../../akka-camel/src/main/resources/reference.conf
   :language: none

.. _config-akka-cluster:

akka-cluster
~~~~~~~~~~~~

.. literalinclude:: ../../../akka-cluster/src/main/resources/reference.conf
   :language: none

.. _config-akka-multi-node-testkit:

akka-multi-node-testkit
~~~~~~~~~~~~~~~~~~~~~~~

.. literalinclude:: ../../../akka-multi-node-testkit/src/main/resources/reference.conf
   :language: none

.. _config-akka-persistence:

akka-persistence
~~~~~~~~~~~~~~~~

.. literalinclude:: ../../../akka-persistence/src/main/resources/reference.conf
   :language: none

.. _config-akka-remote:

akka-remote
~~~~~~~~~~~

.. literalinclude:: ../../../akka-remote/src/main/resources/reference.conf
   :language: none

.. _config-akka-testkit:

akka-testkit
~~~~~~~~~~~~

.. literalinclude:: ../../../akka-testkit/src/main/resources/reference.conf
   :language: none

.. _config-akka-zeromq:

akka-zeromq
~~~~~~~~~~~

.. literalinclude:: ../../../akka-zeromq/src/main/resources/reference.conf
   :language: none


