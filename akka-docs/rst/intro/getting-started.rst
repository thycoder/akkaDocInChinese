入门
===============

先决条件
-------------

Akka要求你的机器安装 `Java 1.6 <http://www.oracle.com/technetwork/java/javase/downloads/index.html>`_ 或更新版本。

入门指南和模板项目
--------------------------------------------

开始学习Akka的最好方式是下载  `Typesafe Activator <http://www.typesafe.com/platform/getstarted>`_
并且尝试一个Akka模板项目。

下载
--------

下载Akka有许多方式。你可以作为Typesafe Platform的一部分来下载它(如上文所述)。你可以下载含微内核的完整发行包，包括所有模块。或者你也可以使用一个构建工具，类似Maven或SBT，从Akka Maven仓库下载依赖。


模块
-------

Akka高度模块化，包含一些JAR包，它们包括不同的功能。

- ``akka-actor`` – 传统的Actor，带类型Actor，IO Actor等等。

- ``akka-agent`` – Agent，集成了Scala STM

- ``akka-camel`` – Akka的Camel集成

- ``akka-cluster`` – 集群成员管理，可伸缩路由。

- ``akka-kernel`` – Akka微内核，用于运行一个基本的微型应用服务器

- ``akka-osgi`` – 在OSGi容器中使用Akka的基本包，包含 ``akka-actor`` 的类

- ``akka-osgi-aries`` – 用于配置Actor系统的Aries蓝图

- ``akka-remote`` – 远程Actor

- ``akka-slf4j`` – SLF4J日志工具(事件总线监听器)

- ``akka-testkit`` – 测试Actor系统的工具箱

- ``akka-zeromq`` – ZeroMQ集成

除了这些稳定的模块，还有一些模块在进入稳定核心的路上，但当前仍然被标注为“试验性”。者并不意味着它们不能正常运行，它主要意思是它们的API还没有足够稳定，进而被认为已冻结。你可以通过向我们的邮件列表提供关于这些模块的反馈，来帮助促进这个过程。


- ``akka-contrib`` – 各种各样的贡献，它们可能会也可能不会被移到核心模块, 参见 :ref:`akka-contrib` 了解更多详情。

实际JAR包文件名的一个例子是 ``@jarName@`` (其他模块类似).

如何查看每个Akka模块的JAR依赖，在 :ref:`dependencies` 一节进行了描述。

Using a release distribution
----------------------------

从http://akka.io/downloads下载所需的发布版本，并且将其解压。

使用一个发布版本
------------------------

Akka夜间快照被发布到http://repo.akka.io/snapshots/，版本号同时包含SNAPSHOT和时间戳。你工作中可以选择一个带时间戳的版本，然后决定何时更新到较新的版本。Akka快照仓库也由http://repo.akka.io/snapshots/代理，它还包含了Akka各模块所依赖的一些其他仓库的代理。

.. warning::

    使用Akka快照，每夜构建，以及里程碑发布版，是不被鼓励的，除非你知道你在做什么。

Microkernel
-----------

Akka发布版本包括微内核。要运行这个微内核，请将你的应用程序jar放到 ``deploy`` 目录，并且使用 ``bin`` 目录中的脚本。

更多详情参见文档中的 :ref:`Microkernel (Scala) <microkernel-scala>` / :ref:`Microkernel (Java) <microkernel-java>`.

.. _build-tool:

使用一个构建工具
------------------

Akka可以跟支持Maven仓库的构建工具一起使用。

Maven 仓库
------------------

对于Akka 2.1-M2以及之后版本：

`Maven Central <http://repo1.maven.org/maven2/>`_

对于之前的Akka版本:

`Akka Repo <http://repo.akka.io/releases/>`_
`Typesafe Repo <http://repo.typesafe.com/typesafe/releases/>`_

在Maven中使用Akka
---------------------

开始使用Akka和Maven的最简单方式是检出 `Typesafe Activator <http://www.typesafe.com/platform/getstarted>`_ 中名为 `Akka Main in Java <http://www.typesafe.com/activator/template/akka-sample-main-java>`_ 的教程。

既然Akka被发布到Maven Central(对于从2.1-M2往后的版本)，将Akka依赖添加到POM是足够的。例如，这里是akka-actor的依赖：

.. code-block:: xml

  <dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-actor_@binVersion@</artifactId>
    <version>@version@</version>
  </dependency>

注：对于快照版本，版本号同时包含SNAPSHOT和时间戳

.. code-block:: xml

    <repositories>
      <repository>
        <id>akka-snapshots</id>
          <snapshots>
            <enabled>true</enabled>
          </snapshots>
        <url>http://repo.akka.io/snapshots/</url>
      </repository>
    </repositories>

**注意**: 对于快照版本，版本号同时包含 ``SNAPSHOT`` 和时间戳。


在SBT中使用Akka
-------------------

开始使用Akka和SBT的最简单方式是检出 `Akka/SBT template <http://www.typesafe.com/resources/getting-started/typesafe-stack/downloading-installing.html#template-projects-for-scala-akka-and-play>`_ 项目。

在SBT中使用Akka的必需知识部分的摘要:

SBT安装指南在 `https://github.com/harrah/xsbt/wiki/Setup <https://github.com/harrah/xsbt/wiki/Setup>`_

``build.sbt`` 文件:

.. parsed-literal::

    name := "My Project"

    version := "1.0"

    scalaVersion := "@scalaVersion@"

    libraryDependencies +=
      "com.typesafe.akka" %% "akka-actor" % "@version@" @crossString@

**注意**: 上面所设置的libraryDependencies特定于SBT v0.12.x或更高版本。如果你使用更老版本的SBT，则libraryDependencies应当如下所示：  

.. parsed-literal::

    libraryDependencies +=
      "com.typesafe.akka" % "akka-actor_@binVersion@" % "@version@"

对于快照版本，快照仓库也需要被添加:

.. parsed-literal::

    resolvers += "Akka Snapshot Repository" at "http://repo.akka.io/snapshots/"


在Gradle中使用Akka
----------------------

至少需要 `Gradle <http://gradle.org>`_ 1.4
使用 `Scala plugin <http://gradle.org/docs/current/userguide/scala_plugin.html>`_

.. parsed-literal::

    apply plugin: 'scala'

    repositories {
      mavenCentral()
    }

    dependencies {
      compile 'org.scala-lang:scala-library:@scalaVersion@'
    }

    tasks.withType(ScalaCompile) {
      scalaCompileOptions.useAnt = false
    }

    dependencies {
      compile group: 'com.typesafe.akka', name: 'akka-actor_@binVersion@', version: '@version@'
      compile group: 'org.scala-lang', name: 'scala-library', version: '@scalaVersion@'
    }

对于快照版本，快照仓库也需要被添加:

.. parsed-literal::

    repositories {
      mavenCentral()
      maven {
        url "http://repo.akka.io/snapshots/"
      }
    }


在Eclipse中使用Akka
-----------------------

建立SBT项目，然后使用 `sbteclipse <https://github.com/typesafehub/sbteclipse>`_ 来生成一个Eclipse项目。

在IntelliJ IDEA中使用Akka
-----------------------------

建立SBT项目，然后使用 `sbt-idea <https://github.com/mpeltonen/sbt-idea>`_ 来生成一个 IntelliJ IDEA项目.

在NetBeans中使用Akka
------------------------

建立SBT项目，然后使用 `nbsbt <https://github.com/dcaoyuan/nbsbt>`_ 来生成一个 NetBeans项目.

你应当使用 `nbscala <https://github.com/dcaoyuan/nbscala>`_ 在IDE中生成scala支持。

不要使用 -optimize 这一Scala编译器标记
----------------------------------------

.. warning::

  Akka没有使用-optimize这一Scala编译器标记来编译或测试。使用它的用户曾经报告一些奇怪的现象。


从源码构建
------------------

Akka使用Git，并且被托管在 `Github <http://github.com>`_.

* Akka: 克隆Akka仓库的地址是 `<http://github.com/akka/akka>`_

接下来阅读 :ref:`building-akka`

需要帮助?
----------

如果你有问题，可以求助 `Akka Mailing List <http://groups.google.com/group/akka-user>`_.

你还可以求助 `commercial support <http://www.typesafe.com>`_.

感谢你成为Akka社区的一员。

