Docker 源码分析之容器启动
===

# 前言
本文基于Docker1.12版本对Docker容器启动过程进行初步的分析。由于作者是初次尝试分析Docker源码，如有错误之处欢迎批评指正。

# 总览

![](http://172.30.40.21/blog/images/docker-source/container-start/docker-architecture.png)

+ Docker后台分为4个进程：docker daemon、docker-containerd、docker-containerd-shim和docker-runc。Docker容器的创建需要这4个进程配合完成。

+ docker daemon作为系统驻留进程，负责对外提供HTTP API接口，收到容器创建请求命令后，通过gRPC方式调用docker-containerd进程。

+ 关于docker-containerd的说明：
```
containerd is a daemon to control runC, built for performance and density. 
containerd leverages runC's advanced features such as seccomp and user namespace support as well as checkpoint and restore for cloning and live migration of containers.
```
+ docker-containerd创建docker-containerd-shim进程，docker-containerd-shim创建docker-runc进程。
+ docker-runc负责最终Docker容器的创建、Namespace的设置，以及容器用户进程的启动。
下面分别进行介绍。

# docker daemon
## 源码解读
docker deamon启动时，将会初始化各种http消息路由器。

![](http://172.30.40.21/blog/images/docker-source/container-start/3.1-1.png)

其中，容器的路由器初始化代码在docker/server/router/container/container.go定义：

![](http://172.30.40.21/blog/images/docker-source/container-start/3.1-2.png)

针对HTTP URL的不同，定义了对应的处理函数。
容器启动对应的处理函数是：r.postContainersStart。
继续看一下r.postContainersStart函数定义：

![](http://172.30.40.21/blog/images/docker-source/container-start/3.1-3.png)

在这里，调用了backend.ContainerStart方法。
backend是啥东西？
在daemon.go的
```go
func (cli *DaemonCli) start() (err error) {
```
方法中有相应的初始化动作：
```go
d, err := daemon.NewDaemon(cli.Config, registryService, containerdRemote)
```
该方法的签名如下：
```go
func NewDaemon(config *Config, registryService registry.Service, containerdRemote libcontainerd.Remote) (daemon *Daemon, err error) {
```
传入了libcontainerd.Remote，返回的是Daemon指针。

libcontainerd.Remote的初始化代码如下：
```go
containerdRemote, err := libcontainerd.New(cli.getLibcontainerdRoot(), cli.getPlatformRemoteOptions()...)
```
这个在后面会用到，具体用户再详细分析。

继续看Daemon对象的ContainerStart方法（在/docker/daemon/start.go代码中定义）：
```go
return daemon.containerStart(container)
```
在containerStart方法里，调用了containerd.Create方法：
```go
if err := daemon.containerd.Create(container.ID, *spec, createOptions...); err != nil {
```
daemon.containerd是libcontainerd.Client接口，该接口定义如下：

![](http://172.30.40.21/blog/images/docker-source/container-start/3.1-4.png)

具体实现类在docker/libcontainerd/client_linux.go代码中定义。
```go
type client struct {
       clientCommon

       // Platform specific properties below here.
       remote        *remote
       q             queue
       exitNotifiers map[string]*exitNotifier
       liveRestore   bool
}

func (clnt *client) Create(containerID string, spec Spec, options ...CreateOption) (err error)
```
在Create方法中，调用newContainer，返回libcontainerd.container对象实例。
```go
container := clnt.newContainer(filepath.Join(dir, containerID), options...)
```
接下来，调用容器的start方法：
```go
return container.start()
```
在容器的启动方法中，先构造容器创建请求：
```go
r := &containerd.CreateContainerRequest{
       Id:         ctr.containerID,
       BundlePath: ctr.dir,
       Stdin:      ctr.fifo(syscall.Stdin),
       Stdout:     ctr.fifo(syscall.Stdout),
       Stderr:     ctr.fifo(syscall.Stderr),
       // check to see if we are running in ramdisk to disable pivot root
       NoPivotRoot: os.Getenv("DOCKER_RAMDISK") != "",
       Runtime:     ctr.runtime,
       RuntimeArgs: ctr.runtimeArgs,
}
```
再调用apiClient执行：
```go
resp, err := ctr.client.remote.apiClient.CreateContainer(context.Background(), r)
```
此处的apiClient定义如下：

![](http://172.30.40.21/blog/images/docker-source/container-start/3.1-5.png)

该接口的功能是封装命令请求，通过gRPC的方式将消息发送给docker-containerd进程。
该接口的实现类在：docker/containerd/api/grpc/types/api.pb.go

![](http://172.30.40.21/blog/images/docker-source/container-start/3.1-6.png)

可以看到，这里使用grpc向docker-containerd发送创建容器请求。

## 主要的类图

![](http://172.30.40.21/blog/images/docker-source/container-start/3.1-docker-daemon-class.png)

## 时序图

![](http://172.30.40.21/blog/images/docker-source/container-start/3.1-docker-daemon-sequence.png)

# docker-containerd

## 源码解读
docker-containerd启动代码在docker/containerd/containerd/main.go定义。
在docker-containerd启动时，主要的代码如下：

![](http://172.30.40.21/blog/images/docker-source/container-start/4.1-1.png)

先创建Supervisor，然后为Supervisor创建10个Worker线程，之后启动Supervisor。
Supervisor的结构定义如下：

![](http://172.30.40.21/blog/images/docker-source/container-start/4.1-2.png)

注意这里有两个channel：startTasks和tasks。
其中，tasks是在Supervisor的Start()方法中处理：

![](http://172.30.40.21/blog/images/docker-source/container-start/4.1-3.png)

如果是容器启动任务，还会构造一个新的startTask，并传递给startTasks channel。

![](http://172.30.40.21/blog/images/docker-source/container-start/4.1-4.png)

Supervisor.worker的Start方法中，读取startTasks channel，并调用runtime.Container接口的Start方法。

![](http://172.30.40.21/blog/images/docker-source/container-start/4.1-5.png)

![](http://172.30.40.21/blog/images/docker-source/container-start/4.1-6.png)

runtime.Container接口的实现类在/docker/containerd/runtime/container.go定义：

![](http://172.30.40.21/blog/images/docker-source/container-start/4.1-7.png)

在其Start()方法中，通过exec.Command创建启动docker-containerd-shim进程的命令，并传入容器ID、bundle和runtime：
```
docker-containerd-shim 817c43b3f5794d0e5dfdb92acf60fe7653b3efc33a4388733d357d00a8d8ae1a /var/run/docker/libcontainerd/817c43b3f5794d0e5dfdb92acf60fe7653b3efc33a4388733d357d00a8d8ae1a docker-runc
```
在createCmd命令中，调用cmd.Start()系统命令启动docker-containerd-shim进程。

![](http://172.30.40.21/blog/images/docker-source/container-start/4.1-8.png)

## 主要的类图

![](http://172.30.40.21/blog/images/docker-source/container-start/4-class.png)

## 时序图

![](http://172.30.40.21/blog/images/docker-source/container-start/4-sequence.png)

# docker-containerd-shim

## 源码解读
docker-containerd-shim的入口在：/docker/containerd/containerd-shim/main.go
根据传入的参数，构造一个process：
```go
p, err := newProcess(flag.Arg(0), flag.Arg(1), flag.Arg(2))
```
创建完毕后，调用process.create()执行进程创建：
```go
if err := p.create(); err != nil
```
process.create()的主要代码如下：

![](http://172.30.40.21/blog/images/docker-source/container-start/5.1-1.png)

构造进程支持参数，主要是create --bundle …  --console …  --pid-file …
构造执行命令：exec.Command(p.runtime, args…)
启动命令：cms.Start()

这里的p.runtime为docker-runc。Args为容器创建参数。
	
# docker-runc
## 源码解读
docker-runc的入口在：/opencontainers/runc/main.go
主要的代码：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-1.png)

这里定义了支持的命令集合。根据传入的参数匹配对应的命令。
容器启动时，传入的是create指令，对应createCommand。
createCommand内定义了对应的Action：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-2.png)

startContainer的逻辑：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-3.png)

先通过createContainer创建一个container对象，初始化runner对象，调用runner的run方法。

createContainer方法的主要逻辑：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-4.png)

先load一个Factory，这里是LinuxFactory。再调factory的Create方法创建容器。
重点看一下loadFactory里的代码：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-5.png)

构造一个LinuxFactory，调用InitArgs进行参数的初始化。

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-6.png)

这里对factory的InitPath和InitArgs进行了赋值。
InitPath是/proc/self/exe，这是一个特殊的程序，如果一个进程调用了它，相当于再次调用此进程。
InitArgs参数为docker-runc的绝对路径，以及init字符串。

LinuxFactory.Create()中构造了linuxContainer对象：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-7.png)

再回到一开始createCommand的startContainer方法，这里调用了runner对象的run方法。该方法的主要逻辑：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-8.png)

调用了container的Run方法。
再看看linuxContainer的Run方法,
代码位于/opencontainer/runc/libcontainer/container_linux.go：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-9.png)

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-10.png)

c.newParentProcess()的主要过程：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-11.png)

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-12.png)

c.newParentProcess()的返回结果是initProcess，其start方法如下：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-13.png)

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-14.png)

主要的流程逻辑：
调用p.cmd.Start()。这里的cmd为：/proc/self/exe path_to_runc init，也即调用docker-runc自身，传入path_to_runc init参数。
由于在main_unix.go中定义了cgo代码，在执行p.cmd.Start()之前，会先调用nsexec.c代码中的nsexec方法。nsexec.c位于opencontainers/runc/libcontainer/nsenter包中，github上对nsenter的描述原文：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-15.png)

nsexec会通过管道从父进程获取bootstrap data（namespace paths, clone flags, uid/gid mapping, console path），clone一个子进程，返回子进程ID给nsexec的父进程。
nsexec执行完毕后，p.cmd.Start()执行，即调用docker-runc path_to_runc init，并通过管道传递config信息。

继续看runc的initCommand：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-16.png)

LinuxFactory.StartInitialization()的主要逻辑：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-17.png)

通过管道解析父进程传递的config数据。
调用linuxStandardInit.init()方法，该方法的主要流程：
```go
设置容器网络：
if err := setupNetwork(l.config); err != nil
设置容器的root路径：
if err := setupRootfs(l.config.Config, console, l.pipe); err != nil
设置容器的主机名：
if err := syscall.Sethostname([]byte(hostname)); err != nil
```

最后启动用户进程：

![](http://172.30.40.21/blog/images/docker-source/container-start/6.1-18.png)



## 时序图

![](http://172.30.40.21/blog/images/docker-source/container-start/6-sequence.png)
