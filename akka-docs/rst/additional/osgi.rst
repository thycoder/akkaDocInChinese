在 OSGi 环境中使用Akka
============

配置 OSGi 框架
------------------------------

要在 OSGi 环境中使用Akka,  ``org.osgi.framework.bootdelegation`` 属性必须被配置为总是将 ``sun.misc`` 包委托到 boot 类加载器，而不是在普通的OSGi类空间中解析。

Activator
---------

要再OSGi环境中引导Akka，你可以使用 ``akka.osgi.ActorSystemActivator`` 类来很方便地搭建ActorSystem。

.. includecode:: code/docs/osgi/Activator.scala#Activator


``ActorSystemActivator`` 用来创建actor系统的类加载器会从应用程序bundle和所有传递依赖中找到资源 (``reference.conf`` 文件) 和类。

``ActorSystemActivator`` 类包含于 ``akka-osgi`` 组件 ::

  <dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-osgi_@binVersion@</artifactId>
    <version>@version@</version>
  </dependency>


示例
------

一个完整的示例项目提供于 `akka-sample-osgi-dining-hakkers <@github@/akka-samples/akka-sample-osgi-dining-hakkers>`_.
 
