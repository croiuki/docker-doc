Docker Web Shell 实现原理
===
#说明
Docker Web Shell实现从Web浏览器以类似SSH的方式登录并操作Docker容器。

#界面展示
![](https://github.com/croiuki/docker-doc/blob/master/images/docker-web-shell/web-shell-1.png)
![](https://github.com/croiuki/docker-doc/blob/master/images/docker-web-shell/web-shell-2.png)
![](https://github.com/croiuki/docker-doc/blob/master/images/docker-web-shell/web-shell-3.png)
Docker Web Shell提供类似传统SSH终端的用户体验。

#实现原理
##概述
![](https://github.com/croiuki/docker-doc/blob/master/images/docker-web-shell/docker-web-shell.png)
主要的系统组件包括：Web浏览器、Docker Controller、Docker Daemon和Docker容器。
+ Web浏览器负责界面呈现。
+ Docker Controller是Docker容器应用的控制中心，作为桥梁，负责消息的转发。
+ Docker Daemon提供HTTP API接口给外部系统调用以访问容器内部。、

##链路建立
Web浏览器运行JS脚本，通过Web Socket与Docker Controller建立通信链路。
Docker Controller通过Docker HTTP API与Docker Daemon建立通信链路。这里使用到Docker HTTP API的Exec Create、Exec Start和Exec Resize接口，并通过hijacking技术，利用Exec Start返回的数据流承载Docker Controller和Docker Daemon之间的交互数据。

##交互
链路建立后，用户就可以在Web浏览器输入字符与Docker容器交互。
当用户在浏览器进行键盘操作时，数据首先通过Web Socket通道到达Docker Controller，Docker Controller通过HTTP API接口透传给Docker Daemon，Docker Daemon再传递给容器。
容器收到数据后，再根据数据代表的含义，按照数据来时的通路，将对应的数据回显到浏览器上。
通过这种方式，就完成了交互的操作。

