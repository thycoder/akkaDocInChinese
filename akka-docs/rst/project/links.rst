.. _support:

#########
 项目
#########

商业支持
^^^^^^^^^^^^^^^^^^

商业支持提供者为 `Typesafe <http://www.typesafe.com>`_.
Akka 属于 `Typesafe Reactive Platform <http://www.typesafe.com/platform>`_.

Mailing List
^^^^^^^^^^^^

`Akka User Google Group <http://groups.google.com/group/akka-user>`_

`Akka Developer Google Group <http://groups.google.com/group/akka-dev>`_


下载
^^^^^^^^^

`<http://akka.io/downloads>`_


源代码
^^^^^^^^^^^

Akka 使用 Git 并托管于 `Github <http://github.com>`_.

* Akka: clone Akka 仓库的地址为 `<http://github.com/akka/akka>`_


发布版仓库
^^^^^^^^^^^^^^^^^^^

所有Akka发布版本都通过Sonatype发布到Maven中央仓库，参见
`search.maven.org
<http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.typesafe.akka%22>`_

快照仓库
^^^^^^^^^^^^^^^^^^^^

每日构建位于 http://repo.akka.io/snapshots/ ，同时具有 ``SNAPSHOT`` 和带时间戳的版本。

对于带时间戳的版本，挑选时间戳的形式为 http://repo.akka.io/snapshots/com/typesafe/akka/akka-actor_@binVersion@/.
属于同一构建的所有Akka模块都具有相同的时间戳。

快照仓库的sbt定义
-------------------------------------

确保将这一仓库添加到sbt resolvers ::

  resolvers += "Typesafe Snapshots" at "http://repo.akka.io/snapshots/"

使用时间戳作为库依赖的版本。例如 ::

    libraryDependencies += "com.typesafe.akka" % "akka-remote_@binVersion@" % 
      "2.1-20121016-001042"

快照仓库的maven定义
---------------------------------------

确保在pom.xml中将此仓库添加到maven仓库列表中 ::

  <repositories>
    <repository>
      <id>akka-snapshots</id>
      <name>Akka Snapshots</name>
      <url>http://repo.akka.io/snapshots/</url>
      <layout>default</layout>
    </repository>
  </repositories>  

使用时间戳作为库依赖的版本。例如 ::

  <dependencies>
    <dependency>
      <groupId>com.typesafe.akka</groupId>
      <artifactId>akka-remote_@binVersion@</artifactId>
      <version>2.1-20121016-001042</version>
    </dependency>
  </dependencies>



