---
layout: default
title: 学习 ovirt 代码（web 服务端部分）
---

## {{ page.title }}

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

* ovirt engine 这个 web 服务器是如何在 frontend 模块中实现和浏览器通信的？

  通过 [gwt-rpc](http://www.gwtproject.org/doc/latest/tutorial/RPC.html)，这里面有张图：

  ![](/images/2015/gwt-rpc_AnatomyOfServices.png)

  其中，左侧是浏览器的 javascript，右侧是 web 服务器中运行着的 java 程序，至于 javascript 是如何访问服务器中的 java 程序的，是由 gwt-rpc 来操心的事情，我们只需要实现图中标示了“Written by you”的几个类就可以了。然后，在 ovirt engine 的代码中经过一番寻找，即可以对号入座：

  * StockPriceService：对应 frontend/webadmin/modules/frontend/src/main/java/org/ovirt/engine/ui/frontend/gwtservices/GenericApiGWTService.java
  * StockPriceServiceAsync：源码没有，但是会在编译的时候自动根据 StockPriceService 生成，具体可以去研究 maven 如何编译 frontend 模块的
  * StockPriceServiceImpl：对应 frontend/webadmin/modules/frontend/src/main/java/org/ovirt/engine/ui/frontend/server/gwt/GenericApiGWTServiceImpl.java

  由于 GenericApiGWTServiceImpl 是运行在 web 服务器中，即运行在 ovirt engine 的 JVM 实例中的，所以它可以调用 web 服务端的各种资源了。稍微浏览一下这个类，就可以发现它使用服务端资源的切入点：

  ~~~ java
  // snip
  private BackendLocal backend;

  @EJB(beanInterface = BackendLocal.class,
          mappedName = "java:global/engine/bll/Backend!org.ovirt.engine.core.common.interfaces.BackendLocal")
  public void setBackend(BackendLocal backend) {
      this.backend = backend;
  }
  ~~~

* GenericApiGWTServiceImpl 怎么调用 web 服务端的资源的呢？

  从上面的代码可知道，GenericApiGWTServiceImpl 使用了一个 EJB。

  先说说 EJB，EJB 是 JavaEE 中的一个[规范](https://www.jcp.org/en/jsr/detail?id=345)。这里有一个非官方的[介绍](http://blog.csdn.net/jojo52013145/article/details/5783677)，它的初衷是有一个用 Java 运行的客户端程序，客户端程序可以远程（通过网络）直接调用运行在服务器端的 Java 类的方法。

  但是在 ovirt engine 里，客户端程序是用 Javascript 运行的，然后通过运行在 web 服务器上的 Java 程序（GenericApiGWTServiceImpl）来调用 EJB，这样，EJB 和调用 EJB 的 Java 程序处于同一个 JVM。不确定这样做有没有什么其它的考虑，但是有一点还是没问题的：EJB 把 web 服务端的资源模块化了，因为在 ovirt engine 里，EJB 不光被以 GWT 为基础的浏览器调用，还被 rest api 模块调用了，还有可能今后会添加其它的模块，来使用 EJB。

  从上面的代码中可以看到，GenericApiGWTServiceImpl 使用了 BackendLocal 这个 EJB，而上面一行的 ```@EJB``` 标注的意思是：BackendLocal 通过其 JNDI 名称（```@EJB``` 标注中的 mappedName 值）将 BackendLocal 的实现赋予 GenericApiGWTServiceImpl。

  JNDI 只是一个根据字符串寻找 EJB 的途径，具体使用方法请自行学习。在代码中搜寻 BackendLocal 的 JNDI 名称，容易找到其实现为 Backend 这个类。从 Backend 类的标注中可以看出，它是一个单例模式（@Singleton），它依赖其它的 EJB（@DependsOn("LockManager")），它会在 web 服务启动的时候马上初始化（@Startup），在它初始化后还会执行 ```create()``` 方法（@PostConstruct），等等。从这个 ```create()``` 方法开始，就算是正式进入 ovirt engine 的 web 服务端代码了，这里不再展开细讲，我只希望本篇文字能够给大家一个看代码的方向。

  总结下从浏览器到 EJB 的过程：

  **浏览器（Javascript）**==GWT-RPC==> **GenericApiGWTServiceImpl（web 服务端的 java 程序）**==JNDI==> **Backend（EJB）**

* Backend 的 Command 和 Query 机制

  Backend 对外提供的接口并不多，可以在 BackendLocal 类里面看出来。从中可以抽象出两类方法：```runQuery()``` 和 ```runXXXAction()```。

  * runAction 方法提供执行任何操作的入口，这里的操作一般都会对 web 服务器或者 web 服务器所管理的虚拟化资源进行一定的改动，比如对 web 服务器的数据库写入操作，对虚拟化资源的操作如创建一台虚拟机。

    在 runAction 运行时，会从参数中获得 Action 的类型，然后通过 java 的反射机制构建出一个对应 Action 类型的 Command 类，然后，所有与该 Action 的逻辑都在这个对应的 Command 里了。例如，在添加数据中心时，前端浏览器要执行一个 Action 的情况，如果看了前一篇文字，就知道这部分的代码在 MVP 框架的 Model 层中：DatacenterListModel：

    ~~~ java
    // snip
    if (model.getIsNew()) {
        // When adding a data center use sync action to be able present a Guide Me dialog afterwards.
        Frontend.getInstance().runAction(VdcActionType.AddEmptyStoragePool,
            new StoragePoolManagementParameter(dataCenter),
            // snip
            );
    }
    ~~~

    从 runAction 方法中可知，Action 为 ```AddEmptyStoragePool```，这个参数被 gwt-rpc 传到 Backend 后，于是通过反射构建出一个 AddEmptyStoragePoolCommand 类出来。查看该类，就知道这个 Action 做了什么了。

  * runQuery 方法提供执行任何查寻的入口，即不会对任何东西进行修改，只是查看，一般都是针对 web 服务器的数据库的读取操作。其调用过程和 runAction 基本一致。

  再总结下从浏览器到 web 服务器端的过程：

  **Model** ==runAction/Query(action/queryType)==> **Frontend** ==GWT-RPC==> **GenericApiGWTServiceImpl（web 服务端的 java 程序）**==JNDI==> **Backend（EJB）**==runAction/Query(action/queryType)==> **action/queryCommand**
