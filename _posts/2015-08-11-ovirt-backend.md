---
layout: default
title: 从 backport ovirt 的主机设备穿透代码学习 ovirt 代码（web 服务端部分）
---

## {{ page.title }}

### 先了解 ovirt engine 的 web 服务器端

说 ovirt engine 是一个虚拟化的管理平台，其实主要是因为有了 web 服务端，它把相对于比较底层的对虚拟机的操作进行了封装，或者说流程化。使得用户在界面上的操作简洁化。

ovirt 的 web 服务端运行在一个 linux 系统环境中，一般运行在 fedora 或 centos 上。我们来一步一步深入刨析 web 服务端的运行过程。

* 当从 ovirt 官网的[快速开始手册](http://www.ovirt.org/Quick_Start_Guide)搭建好 ovirt 环境，执行 ```ps aux |grep ovirt-engine``` 时，会看到类似如下的输出：

  ```
  ovirt     1473  4.6 18.3 3979644 1502444 ?     Sl   Jul28 925:34 ovirt-engine -server -XX:+TieredCompilation -Xms1g -Xmx1g -XX:PermSize=256m -XX:MaxPermSize=256m -Djava.net.preferIPv4Stack=true -Dsun.rmi.dgc.client.gcInterval=3600000 -Dsun.rmi.dgc.server.gcInterval=3600000 -Djava.awt.headless=true -Djsse.enableSNIExtension=false -Djava.security.krb5.conf=/etc/ovirt-engine/krb5.conf -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/ovirt-engine/dump -Djava.util.logging.manager=org.jboss.logmanager -Dlogging.configuration=file:///var/lib/ovirt-engine/jboss_runtime/config/ovirt-engine-logging.properties -Dorg.jboss.resolver.warning=true -Djboss.modules.system.pkgs=org.jboss.byteman -Djboss.modules.write-indexes=false -Djboss.server.default.config=ovirt-engine -Djboss.home.dir=/usr/share/ovirt-engine-jboss-as -Djboss.server.base.dir=/usr/share/ovirt-engine -Djboss.server.data.dir=/var/lib/ovirt-engine -Djboss.server.log.dir=/var/log/ovirt-engine -Djboss.server.config.dir=/var/lib/ovirt-engine/jboss_runtime/config -Djboss.server.temp.dir=/var/lib/ovirt-engine/jboss_runtime/tmp -Djboss.controller.temp.dir=/var/lib/ovirt-engine/jboss_runtime/tmp -jar /usr/share/ovirt-engine-jboss-as/jboss-modules.jar -mp /var/lib/ovirt-engine/jboss_runtime/modules/00-ovirt-engine-modules:/var/lib/ovirt-engine/jboss_runtime/modules/01-ovirt-engine-jboss-as-modules -jaxpmodule javax.xml.jaxp-provider org.jboss.as.standalone -c ovirt-engine.xml
  ```

* 分析上面的进程信息，可以知道 ovirt engine 的 web 服务端是一个 JVM 实例，并能抽象出如下命令：

  ```
  ovirt-engine [jvm 参数 | 系统环境参数（-D开头）] -jar /usr/share/ovirt-engine-jboss-as/jboss-modules.jar [main 函数参数]
  ```

  其中的 ```ovirt-engine``` 相当于 ```java``` 命令，后面跟了一堆 jvm 参数。可以看出来，其实 ovirt engine 一启动只是运行了一个 jar 文件：jboss-modules.jar

* 这个 jar 文件来自于 [jboss-modules](https://docs.jboss.org/author/display/MODULES/Introduction) 项目，它用于管理 jboss 服务容器的启动。对于 jboss，为了让本文不太复杂，这里暂时不展开说了，以后再专门列出文章来解析。
但是现在需要明白的是，jboss-modules.jar 通过前面的系统环境参数和后面的 main 函数参数，最终把 jboss 启动起来，并且将 ```/var/lib/ovirt-engine/jboss_runtime/deployments``` 里的 engine.ear 和 restapi.war 部署到了运行着的 jboss 服务中。

* engine.ear 包含了管理平台的主体，restapi.war 包括了 ovirt engine 的 rest api 服务。这些文件是由 maven 从 ovirt 代码中编译生成的。这里先不说 jboss 是如何使用 ear 和 war 包了，我们把注意力转移到 engine.ear 是哪来的。

说到了 engine.ear，接下来就可以看看它是怎么来的。查看 ovirt engine 的[代码](https://github.com/ovirt/ovirt-engine)，就可以看出来它是一个 maven 类型的 java 工程，因为从代码的跟目录开始，在大多数的路径下都有一个叫作 pom.xml 的文件。只要对 maven 有一些了解就能从代码跟目录开始构建出一个以 maven 管理的 maven 模块树出来，而 engine.ear 就来自于 ear 模块中的 maven-ear-plugin（见 ear/pom.xml），它的作用是对整个 ovirt engine 的 maven 模块进行封装。

比较重要的几个 maven 模块有：

* webadmin：就是前一篇文章中的 UI 部分
* frontend：在 B/S 模式的 ovirt engine 中，负责连接浏览器与服务器的模块
* dal：数据库层，负责与数据库的通信，DAO 什么的都在里面
* vdsbroker：负责ovirt engine 服务器与宿主机的 agent 进行通信的模块，这里，宿主机的 agent 特指 vdsm，vdsbroker 的作用是专门负责管理宿主机，包括宿主机的状态监控，发送指令等，而 vdsm 则是专门处理 vdsbroker 发来的各种指令。可以说，ovirt engine 就是通过与宿主机的通信来完成绝大多数的任务的。
* bll：即 business logic layer，负责各个模块的联系，它连接了 vdsbroker，frontend，dal 模块，可以说是枢纽，主要做的事情就是数据的处理，且可靠地处理（如数据库的事物；各种模块间的协调，如把数据从一个模块的一种表示形式传递给另一个模块的另一种表示形式；不受数据库事物功能控制的错误的处理，如宿主机的错误处理）

以上是从 maven 模块的角度来分析的，有些地方容易让人知其然不知其所以然，所以下面来个更具体的分析，从代码的角度：

* ovirt engine 这个 web 服务器是如何在 frontend 中实现和浏览器通信的？

  通过 [gwt-rpc](http://www.gwtproject.org/doc/latest/tutorial/RPC.html)，这里面有张图：

  ![](/images/2015/gwt-rpc_AnatomyOfServices.png)

  其中，左侧是浏览器的 javascript，右侧是 web 服务器中运行着的 java 程序，至于 javascript 是如何访问 servlet 的，这个是由 gwt-rpc 来操心的事情，我们只需要实现图中标示了“Written by you”的几个类就可以了。然后，在 ovirt engine 的代码中经过一番寻找，即可以对号入座：

  * StockPriceService：对应 frontend/webadmin/modules/frontend/src/main/java/org/ovirt/engine/ui/frontend/gwtservices/GenericApiGWTService.java
  * StockPriceServiceAsync：源码没有，但是会在编译的时候自动根据 StockPriceService 生成，具体可以去研究 maven 如何编译 frontend 模块的
  * StockPriceServiceImpl：对应 frontend/webadmin/modules/frontend/src/main/java/org/ovirt/engine/ui/frontend/server/gwt/GenericApiGWTServiceImpl.java

  由于 GenericApiGWTServiceImpl 是运行在 web 服务器中，即运行在 ovirt engine 的 JVM 实例中的，所以它可以调用 web 服务端的各种资源了。稍微浏览一下这个类，就可以发现使用服务端资源的切入点：

  {% highlight java linenos %}
  // snip
  private BackendLocal backend;

  @EJB(beanInterface = BackendLocal.class,
          mappedName = "java:global/engine/bll/Backend!org.ovirt.engine.core.common.interfaces.BackendLocal")
  public void setBackend(BackendLocal backend) {
      this.backend = backend;
  }
  {% endhighlight %}

* 从上面的代码可知道，GenericApiGWTServiceImpl 使用了一个 EJB。EJB 是 JavaEE 中的一个[规范](https://www.jcp.org/en/jsr/detail?id=345)。这里有一个非官方的[介绍](http://blog.csdn.net/jojo52013145/article/details/5783677)
