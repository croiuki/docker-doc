使用Macvlan构建Docker网络
===

# 简介

- Macvlan是Linux内核支持的网络接口。要求的Linux内部版本是v3.9–3.19 和 4.0+。
- 通过为物理网卡创建Macvlan子接口，允许一块物理网卡拥有多个独立的MAC地址和IP地址。虚拟出来的子接口将直接暴露在底层物理网络中。从外界看来，就像是把网线分成多股，分别接到了不同的主机上一样。
- Macvlan有四种工作模式：Private、VEPA、Bridge和Passthru。最常用和默认的模式是Bridge模式。具体这几种模式的差别参见：
http://blog.csdn.net/dog250/article/details/45788279

- 物理网卡收到包后，会根据收到包的目的 MAC 地址判断这个包需要交给哪个虚拟网卡：

![](http://172.30.40.21/blog/images/docker-macvlan/macvlan-workmode-1.png)

- 如果配合Network Namespace 使用，可以构建这样的网络：

![](http://172.30.40.21/blog/images/docker-macvlan/macvlan-workmode-namespace.png)

# 使用Macvlan构建Docker网络

## 网络模型
![](http://172.30.40.21/blog/images/docker-macvlan/macvlan-docker.png)

通过Macvlan创建的子接口和网卡同属一个子网。

## 操作步骤
- 1.使用docker network命令创建类型为macvlan的Docker网络macvlan_pub：
```
	docker network create -d macvlan \
    	--subnet=192.168.51.0/24 \
    	--gateway=192.168.51.254  \
    	-o parent=enp0s3 macvlan_pub
```

- 2.使用macvlan_pub，并指定IP运行容器：
```
	docker  run --net=macvlan_pub --ip=192.168.51.121 -itd alpine /bin/sh
	docker  run --net=macvlan_pub --ip=192.168.51.122 -itd alpine /bin/sh
```

- 3.登录其中一个容器查看IP：
```
	/ # ip addr show eth0
	11: eth0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether 02:42:c0:a8:33:79 brd ff:ff:ff:ff:ff:ff
    inet 192.168.51.121/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:c0ff:fea8:3379/64 scope link 
       valid_lft forever preferred_lft forever
```

- 4.查看容器路由：
```
	/ # route -n
	Kernel IP routing table
	Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
	0.0.0.0         192.168.51.254  0.0.0.0         UG    0      0        0 eth0
	192.168.51.0    0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

- 5.Ping另一个容器：
```
	/ # ping 192.168.51.122
	PING 192.168.51.122 (192.168.51.122): 56 data bytes
	64 bytes from 192.168.51.122: seq=0 ttl=64 time=0.140 ms
	64 bytes from 192.168.51.122: seq=1 ttl=64 time=0.127 ms
```
Ping网关：
```
	/ # ping 192.168.51.254
	PING 192.168.51.254 (192.168.51.254): 56 data bytes
	64 bytes from 192.168.51.254: seq=0 ttl=255 time=1.669 ms
	64 bytes from 192.168.51.254: seq=1 ttl=255 time=1.655 ms
```

- 6.在另一台宿主机上重复（1）-（5）步骤创建Docker网络，启动容器。两台宿主机上的Docker容器都可以互相Ping通。

## Linux操作指令
设置网卡enp0s3为混杂模式：
```
	ifconfig enp0s3 promisc
```
为物理网卡enp0s3创建两个Macvlan子接口：
```
	sudo ip link add mymacvlan1 link enp0s3 type macvlan mode bridge
	sudo ip link add mymacvlan2 link enp0s3 type macvlan mode bridge
	sudo ip link set mymacvlan1 up
	sudo ip link set mymacvlan2 up
```
可以看到，两个Macvlan子接口的MAC地址和父接口enp0s3的都不一样：
```
	[root@centos-2 ~]# ip link show mymacvlan1 
	15: mymacvlan1@enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT 
    link/ether 6e:de:cc:b8:20:7f brd ff:ff:ff:ff:ff:ff

	[root@centos-2 ~]# ip link show mymacvlan2
	16: mymacvlan2@enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT 
    link/ether 22:7b:b0:89:fa:2b brd ff:ff:ff:ff:ff:ff

	[root@centos-2 ~]# ip link show enp0s3
	2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
    link/ether 08:00:27:9a:47:3e brd ff:ff:ff:ff:ff:ff 
```

# 使用Macvlan+VLAN构建Docker网络

## VLAN简介
- VLAN（Virtual Local Area Network）又称虚拟局域网，是指在局域网的基础上，采用网络管理软件构建的可跨越不同网段、不同网络的端到端的逻辑网络。
- 一个VLAN组成一个逻辑子网，即一个逻辑广播域，它可以覆盖多个网络设备，允许处于不同地理位置的网络用户加入到一个逻辑子网中。
- 使用VLAN功能后，能够将网络分割成多个广播域。
- Linux支持在物理网卡上创建vlan子接口。每个vlan子接口属于不同的二层域，所有的vlan子接口拥有相同的MAC地址。这点是和Macvlan子接口不同的地方。

## 网络模型
![](http://172.30.40.21/blog/images/docker-macvlan/macvlan+vlan-docker.png)

## 操作步骤
- 1.在宿主机网卡enp0s3上创建两个vlan子接口：
```shell
    ip link add link enp0s3 name enp0s3.200 type vlan id 200
	ip link set enp0s3.200 up
	ip link add link enp0s3 name enp0s3.201 type vlan id 201
	ip link set enp0s3.201 up
```

- 2.使用docker network命令创建类型为macvlan的Docker网络macvlan200和macvlan201，父接口指向上一步骤创建的vlan子接口：
```
	docker network  create  -d macvlan \
		--subnet=192.168.200.0/24 \
		--gateway=192.168.200.1 \
		-o parent=enp0s3.200 macvlan200

	docker network  create  -d macvlan \
		--subnet=192.168.201.0/24 \
		--gateway=192.168.201.1 \
		-o parent=enp0s3.201 macvlan201
```

- 3.基于macvlan200创建容器：
```
	docker run --net=macvlan200 -idt --ip=192.168.200.2 alpine /bin/sh
	docker run --net=macvlan200 -idt --ip=192.168.200.3 alpine /bin/sh
```

- 4.基于macvlan201创建容器：
```
	docker run --net=macvlan201 -idt --ip=192.168.201.2 alpine /bin/sh
	docker run --net=macvlan201 -idt --ip=192.168.201.3 alpine /bin/sh
```

- 5.在另一个宿主机上重复（1）-（4）步骤创建基于macvlan200和macvlan201的容器，容器IP分别为：192.168.200.128、192.168.200.129、192.168.201.128、192.168.201.129。


- 6.不同宿主机上，相同vlan的容器可以互通，不同vlan的容器不能互通。如果需要不同vlan的容器互通，需要在路由器上做相应的设置。


- 7.在宿主机1上容器192.168.201.3里Ping宿主机2上容器192.168.201.128，通过抓包可以看到ICMP包中带了802.1Q VLAN Tag：
![](http://172.30.40.21/blog/images/docker-macvlan/macvlan-tcpdump.png)
