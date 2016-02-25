---
layout: default
title: debug oVirt 前端
---

## {{ page.title }}

### oVirt GWT 开发模式

![](/images/2015/debug-ovirt-frontend.png)

在上图中，开发者可以通过在 eclipse IDE 里面修改代码（当然仅限前端代码），然后刷新 firefox 浏览器，
即可看到修改代码后的效果。要实现该开发模式，需要设置如下几个服务：

* 上图中的 gwt debug 服务器

  * 下载 ovirt-engine 代码
  * 开发模式的编译，见代码跟目录的 README.developer 的 BUILD 部分
  * 开启gwt debug 服务：见代码跟目录的 README.developer 的 GWT DEBUG 部分，如：

    ~~~
    make gwt-debug DEBUG_MODULE="webadmin" DEV_EXTRA_BUILD_FLAGS_GWT_DEFAULTS="-Dgwt.userAgent=gecko1_8"
    ~~~

    并看到输出信息中提示正在监听 8000 端口。
  * 上图中的 eclipse remote java application

    * 下载 eclipse for javaee developers
    * 打开 eclipse，并导入 ovirt-engine 代码
    * 新建一个 remote java application：

      * 点击 debug 图标边上的下拉按钮，在下拉菜单中点击 “Debug Configuration” 按钮
      * 新建一个“remote java application”

        * name 随意填写
        * Project 选择 webadmin
        * 在 Connection Properties 下

          * Host 选择 localhost
          * Port 选择 8000
      * 点击 Apply
      * 点击 Debug

        * 这个时候，如果你使用的是 ipv4 的环境，可能 remote java application 默认绑定了 ipv6 的地址，需要设置 ```_JAVA_OPTIONS``` 环境变量来告诉 jvm 默认绑定 ipv4 地址，在 ```~/.bashrc``` 下添加内容： ```export _JAVA_OPTIONS="-Djava.net.preferIPv4Stack=true"``` 即可

    * 顺利的话会弹出一个标题为“GWT Development Mode”的窗口

  * 上图中的 ovirt-engine 管理平台服务器

    * TODO

  * 上图中的 GWT Developer Plugin：在第一次访问 debug 链接的时候浏览器会有提示说要安装[插件](https://dl-ssl.google.com/gwt/plugins/firefox/gwt-dev-plugin.xpi)，详见下面一步

    * 上图中的 firefox <= 24.3.0，[下载地址](https://ftp.mozilla.org/pub/mozilla.org/firefox/releases/24.3.0esr/linux-x86_64/en-US/firefox-24.3.0esr.tar.bz2)。其它浏览器待测试

      * 打开浏览器，然后设置禁用浏览器更新
      * 因为 GWT Developer Plugin 由于维护困难（浏览器众多，版本号众多）不再进行新的开发，目前支持的最新的 firefox 浏览器版本为 24.3。
        GWT 社区现在转而使用 Super Dev Mode， 而 oVirt 目前（3.5）不支持 [Super Dev Mode](http://www.gwtproject.org/articles/superdevmode.html)，所以需要该版本的浏览器。
      * 在浏览器输入合适的 debug 地址，如：```https://192.168.3.167/ovirt-engine/webadmin/WebAdmin.html?gwt.codesvr=127.0.0.1:9997```
        表示：192.168.3.167 为 ovirt-engine 管理平台服务器地址，127.0.0.1 为 eclipse remote java application 的地址

        * 如果 ovirt-engine 管理平台服务器使用的是 https 连接，浏览器默认会阻止访问，因为 eclipse remote java application 的连接没有采用 https，
          浏览器默认不允许在 https 的连接页面里包含 非 https 连接的内容。此时可以：

          * 点击连接最左侧的盾牌安全图标，弹出对话框提示“Firefox has blocked content that isn't secure”
          * 点击对话框右下角下拉项目：“Disable Protection on This Page”
          * 如果是第一次访问

            * 确保浏览器能够访问 google.com  和 www.gwtproject.org
            * 此时会弹出一个页面提示安装 GWT Developer Plugin
            * 安装该插件，然后重启浏览器

### 注意：

* 上图中，eclipse remote java application，gwt debug 服务器，ovirt-engine 管理平台服务器都可以灵活配置，它们既可以在一台机器，也可以在不同的机器上。
* 在实际 debug 的时候，往往是在测试或者生产环境上界面出了问题，这个时候可以在本地的桌面环境搭建 gwt debug 服务器和 eclipse remote java application，然后访问出问题的 ovirt-engine 管理平台环境。
* 运行 gwt debug 服务器和 firefox 浏览器的环境需要较大的内存，建议 8G 以上
