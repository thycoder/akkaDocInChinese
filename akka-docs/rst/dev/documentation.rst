.. highlightlang:: rest

.. _documentation:

#########################
 文档指导方针
#########################

Akka 文档使用 `reStructuredText`_ 作为其标记语言，并且构建时使用了 `Sphinx`_.

.. _reStructuredText: http://docutils.sourceforge.net/rst.html
.. _sphinx: http://sphinx.pocoo.org


Sphinx
======

了解更多信息请参见 `The Sphinx Documentation <http://sphinx.pocoo.org/contents.html>`_

reStructuredText
================

了解更多信息请参见 `The reST Quickref <http://docutils.sourceforge.net/docs/user/rst/quickref.html>`_

小节
--------

在 reST 中小节标题非常灵活. 在Akka文档中我们使用如下惯例:

* ``#`` (上面或下面) 用于模块标题
* ``=`` 用于小节
* ``-`` 用于小小节
* ``^`` 用于小小小节
* ``~`` 用于小小小小节


交叉引用
-----------------

能够在文档中被交叉引用的小节，应该被标记为一个引用。要标记一个小节，请在小节开头之前使用 ``.. _ref-name:`` 。 然后这个小节就可以使用 ``:ref:`ref-name``` 来链接。这些是整个文档中唯一的引用。

例如::

  .. _akka-module:

  #############
   Akka Module
  #############

  这是一个模块文档

  .. _akka-section:

  Akka Section
  ============

  Akka Subsection
  ---------------

  这是一个到 "akka section" 的引用: :ref:`akka-section` 的名字将是 "Akka Section" 。

收件文档
=======================

首先安装 `Sphinx`_. 参见下面。

构建
--------

得到文档的html版本 ::

    sbt sphinx:generateHtml

    open <project-dir>/akka-docs/target/sphinx/html/index.html

得到文档的pdf版本 ::

    sbt sphinx:generatePdf

    open <project-dir>/akka-docs/target/sphinx/latex/AkkaJava.pdf
    或
    open <project-dir>/akka-docs/target/sphinx/latex/AkkaScala.pdf

在 OS X 上安装 Sphinx
-------------------------

安装 `Homebrew <https://github.com/mxcl/homebrew>`_

安装 Python 和 pip:

::

  brew install python
  /usr/local/share/python/easy_install pip

将 Homebrew Python 路径添加到你的 $PATH:

::

  /usr/local/Cellar/python/2.7.5/bin


遇到问题时可以查看更多信息:
https://github.com/mxcl/homebrew/wiki/Homebrew-and-Python

安装Sphinx:

::

  pip install sphinx

将 sphinx_build 添加到你的 $PATH:

::

  /usr/local/share/python

按照 BasicTeX 包:
http://www.tug.org/mactex/morepackages.html

将 texlive bin添加到 $PATH:

::

  /usr/local/texlive/2013basic/bin/universal-darwin

添加丢失的 tex 包:

::

  sudo tlmgr update --self
  sudo tlmgr install titlesec
  sudo tlmgr install framed
  sudo tlmgr install threeparttable
  sudo tlmgr install wrapfig
  sudo tlmgr install helvetic
  sudo tlmgr install courier
  sudo tlmgr install multirow

如果你在生成文档时看到错误 "unknown locale: UTF-8" ，解决方法是定义以下的环境变量:

::

  export LANG=en_US.UTF-8
  export LC_ALL=en_US.UTF-8

