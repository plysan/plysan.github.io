---
layout: default
title: 从 backport ovirt 的主机设备穿透代码学习 ovirt 代码（前端部分）
---

## {{ page.title }}

一直基于开源的虚拟化项目 ovirt 做产品，也渐渐对这个以 java 为核心的管理系统有所熟悉。
前一段时间，公司有需求要实现宿主机的显卡和网卡设备的穿透功能，而当前公司的产品基于的是 ovirt 3.5，碰巧，ovirt 3.6 也实现好了宿主机的设备穿透功能，
拿 3.6 的跑了一遍没有问题，并且 3.5 和 3.6 之间的代码区别还没有大到水火不相容的程度，于是萌生了 backport 的想法，正好从界面到后端纵向的过一遍。

所谓 backport 就是将同一个项目的新版本实现好的代码转移到旧版本中，在一个项目中，对新功能的开发往往都是在该项目的最新版本上进行，而老版本则主要进入维护阶段。

主机设备穿透功能想法很简单，就是将把运行了虚拟机的宿主机上的硬件设备（可为 PCI 设备，usb 设备）穿透给虚拟机使用。
穿透给了虚拟机使用的设备主机不能再使用，并且虚拟机没有经过虚拟化层，而是直接从硬件层面上与宿主机的设备进行通信，在性能，延迟方面自然有先天优势，但是其劣势也是很显然的，它就像反虚拟化，把虚拟化的种种优势给抛开了。例如，一台虚拟机如果穿透了宿主机上的设备，它就只能在这台宿主机上面运行，失去了虚拟机可以在集群中随处运行的灵活性，并且虚拟机是独占式的使用该设备，导致资源的利用率可能不高。不过对于某些场景，该功能还是很实用且必不可少的。

#### ovirt engine 前端

ovirt engine 前端采用了 gwtp 结构，即 gwt 的 MVP 实现，即 Model，View，Presenter。这种模式在当下的很多前端业务逻辑框架中都有采用。

* Model：主要负责逻辑，比如界面元素的内容写入，向后台发送数据请求
* View：顾名思义，即界面元素的样式以及界面结构定义
* Presenter：控制单元，负责控制 View 在什么时候展现，比如用户点击某个按钮弹出窗口，窗口的弹出就是按钮事件所触发的回调函数调用了由该窗口所对应的 Presenter 来弹出的

接下来需要找到管理员门户的前端的 M、V、P 所对应的代码：

* M：在 ovirt 中，Model 的定义都在 uicommonweb 这个 maven 工程里，根据包名很快就能对号入座
* V：对于管理员门户，View 放在了 webadmin 这个 maven 工程里
* P：因为在 ovirt 中，一般一个 View 就对应了一个 Presenter，所以它也放在了 webadmin 中，根据包名很容易区分它们

对于宿主机设备穿透功能，界面上要体现的元素有：

* 主机下的子选项卡应该多出一个选项卡，用于列出主机的所有设备
  ![](/images/2015/host_device_list.png)
* 虚拟机的子选项卡下面应该多出一个选项卡，用于列出虚拟机已经使用（穿透）的主机设备
  ![](/images/2015/vm_host_device_list.png)
* 设置按钮用于编辑用于穿透的主机设备，点击设置按钮后还需要有对应的弹出窗口
  ![](/images/2015/vm_host_device_dialog.png)

根据以上，于是从 ovirt 3.6 的代码中找到了以下文件：

* 虚拟机设置主机设备的弹出窗口所对应的 MVP：

  * frontend/webadmin/modules/uicommonweb/src/main/java/org/ovirt/engine/ui/uicommonweb/models/vms/hostdev/AddVmHostDevicesModel.java
  * frontend/webadmin/modules/webadmin/src/main/java/org/ovirt/engine/ui/webadmin/section/main/view/popup/vm/AddVmHostDevicePopupView.java
  * frontend/webadmin/modules/webadmin/src/main/java/org/ovirt/engine/ui/webadmin/section/main/presenter/popup/hostdev/AddVmHostDevicePopupPresenterWidget.java

* 虚拟机已使用的主机设备列表的子选项卡所对应的 MVP：

  * frontend/webadmin/modules/uicommonweb/src/main/java/org/ovirt/engine/ui/uicommonweb/models/vms/hostdev/VmHostDeviceListModel.java
  * frontend/webadmin/modules/webadmin/src/main/java/org/ovirt/engine/ui/webadmin/section/main/view/tab/virtualMachine/SubTabVirtualMachineHostDeviceView.java
  * frontend/webadmin/modules/webadmin/src/main/java/org/ovirt/engine/ui/webadmin/section/main/presenter/tab/virtualMachine/SubTabVirtualMachineHostDevicePresenter.java

* 主机的设备列表的子选项卡所对应的 MVP：

  * frontend/webadmin/modules/uicommonweb/src/main/java/org/ovirt/engine/ui/uicommonweb/models/vms/hostdev/HostDeviceListModel.java
  * frontend/webadmin/modules/webadmin/src/main/java/org/ovirt/engine/ui/webadmin/section/main/view/tab/host/SubTabHostDeviceView.java
  * frontend/webadmin/modules/webadmin/src/main/java/org/ovirt/engine/ui/webadmin/section/main/presenter/tab/host/SubTabHostDevicePresenter.java

ovirt 前端还采用了 GIN 注入框架，即 google inject，这很像 java 里著名的 shh 框架里的 Spring，用于将面向对象的 Java 语言的对象的创建集中起来，以便于管理，比如某类的对象是单例还是多例。

对于 GWTP 框架来说，它使用了 GIN 作为注入的方式，具体注入的内容有：

1. MVP 之间的关系的绑定：告诉 GWTP，当某一个 Presenter 被调用时，应该使用哪一个 View 和 Model，此关系定义在下面这个文件中：

   * frontend/webadmin/modules/webadmin/src/main/java/org/ovirt/engine/ui/webadmin/gin/PresenterModule.java

   所以在该文件中，需要通过 ```bindPresenter()``` 函数来绑定上面三个 MVP。

   此时，对于主机和虚拟机的子选项卡的 backport 工作已经完成，但对于虚拟机设置主机设备的弹出窗口，还需要额外工作。在继续修改代码前，需要先了解 ovirt 处理弹出窗口的逻辑。

2. ModelPorvider 的定义：用于定义 UICommand 和 Presenter 之间的对应关系，例如在某一个 View 中触发了一个按钮操作，按钮操作会触发一个 UICommand 事件，该 UICommand 会传给指定的 ModelProvider，然后 ModelPorvider 会执行自己的逻辑返回对应的 Presenter，从而将弹出窗口界面显示出来。对于虚拟机的弹出窗口，需要定义 ModelPorvider 的地方为：

   * frontend/webadmin/modules/webadmin/src/main/java/org/ovirt/engine/ui/webadmin/gin/uicommon/VirtualMachineModule.java

   在里面增加 ```getVmHostDeviceListProvider()``` 函数，如下（省略了部分）：

   {% highlight java linenos %}
   public SearchableDetailModelProvider<...> getVmHostDeviceListProvider(...) {
       return new SearchableDetailTabModelProvider<...> (...) {
           public AbstractModelBoundPopupPresenterWidget<...> getModelPopup(...) {
               if (lastExecutedCommand == getModel().getAddCommand()) {
                   return addPopupProvider.get();
               } else if (lastExecutedCommand == getModel().getRepinHostCommand()) {
                   return repinPopupProvider.get();
               }
               return super.getModelPopup(source, lastExecutedCommand, windowModel);
           }
       }
   }
   {% endhighlight %}

至此，ovirt 前端部分基本完成。

{{ page.date | date_to_string }}
