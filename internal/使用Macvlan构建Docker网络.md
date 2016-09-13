ʹ��Macvlan����Docker����
===

# ���

- Macvlan��Linux�ں�֧�ֵ�����ӿڡ�Ҫ���Linux�ڲ��汾��v3.9�C3.19 �� 4.0+��
- ͨ��Ϊ������������Macvlan�ӽӿڣ�����һ����������ӵ�ж��������MAC��ַ��IP��ַ������������ӽӿڽ�ֱ�ӱ�¶�ڵײ����������С�����翴���������ǰ����߷ֳɶ�ɣ��ֱ�ӵ��˲�ͬ��������һ����
- Macvlan�����ֹ���ģʽ��Private��VEPA��Bridge��Passthru����ú�Ĭ�ϵ�ģʽ��Bridgeģʽ�������⼸��ģʽ�Ĳ��μ���
http://blog.csdn.net/dog250/article/details/45788279

- ���������յ����󣬻�����յ�����Ŀ�� MAC ��ַ�ж��������Ҫ�����ĸ�����������

![](http://172.30.40.21/blog/images/docker-macvlan/macvlan-workmode-1.png)

- ������Network Namespace ʹ�ã����Թ������������磺

![](http://172.30.40.21/blog/images/docker-macvlan/macvlan-workmode-namespace.png)

# ʹ��Macvlan����Docker����

## ����ģ��
![](http://172.30.40.21/blog/images/docker-macvlan/macvlan-docker.png)

ͨ��Macvlan�������ӽӿں�����ͬ��һ��������

## ��������
- 1.ʹ��docker network���������Ϊmacvlan��Docker����macvlan_pub��
```
	docker network create -d macvlan \
    	--subnet=192.168.51.0/24 \
    	--gateway=192.168.51.254  \
    	-o parent=enp0s3 macvlan_pub
```

- 2.ʹ��macvlan_pub����ָ��IP����������
```
	docker  run --net=macvlan_pub --ip=192.168.51.121 -itd alpine /bin/sh
	docker  run --net=macvlan_pub --ip=192.168.51.122 -itd alpine /bin/sh
```

- 3.��¼����һ�������鿴IP��
```
	/ # ip addr show eth0
	11: eth0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether 02:42:c0:a8:33:79 brd ff:ff:ff:ff:ff:ff
    inet 192.168.51.121/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:c0ff:fea8:3379/64 scope link 
       valid_lft forever preferred_lft forever
```

- 4.�鿴����·�ɣ�
```
	/ # route -n
	Kernel IP routing table
	Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
	0.0.0.0         192.168.51.254  0.0.0.0         UG    0      0        0 eth0
	192.168.51.0    0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

- 5.Ping��һ��������
```
	/ # ping 192.168.51.122
	PING 192.168.51.122 (192.168.51.122): 56 data bytes
	64 bytes from 192.168.51.122: seq=0 ttl=64 time=0.140 ms
	64 bytes from 192.168.51.122: seq=1 ttl=64 time=0.127 ms
```
Ping���أ�
```
	/ # ping 192.168.51.254
	PING 192.168.51.254 (192.168.51.254): 56 data bytes
	64 bytes from 192.168.51.254: seq=0 ttl=255 time=1.669 ms
	64 bytes from 192.168.51.254: seq=1 ttl=255 time=1.655 ms
```

- 6.����һ̨���������ظ���1��-��5�����贴��Docker���磬������������̨�������ϵ�Docker���������Ի���Pingͨ��

## Linux����ָ��
��������enp0s3Ϊ����ģʽ��
```
	ifconfig enp0s3 promisc
```
Ϊ��������enp0s3��������Macvlan�ӽӿڣ�
```
	sudo ip link add mymacvlan1 link enp0s3 type macvlan mode bridge
	sudo ip link add mymacvlan2 link enp0s3 type macvlan mode bridge
	sudo ip link set mymacvlan1 up
	sudo ip link set mymacvlan2 up
```
���Կ���������Macvlan�ӽӿڵ�MAC��ַ�͸��ӿ�enp0s3�Ķ���һ����
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

# ʹ��Macvlan+VLAN����Docker����

## VLAN���
- VLAN��Virtual Local Area Network���ֳ��������������ָ�ھ������Ļ����ϣ��������������������Ŀɿ�Խ��ͬ���Ρ���ͬ����Ķ˵��˵��߼����硣
- һ��VLAN���һ���߼���������һ���߼��㲥�������Ը��Ƕ�������豸�������ڲ�ͬ����λ�õ������û����뵽һ���߼������С�
- ʹ��VLAN���ܺ��ܹ�������ָ�ɶ���㲥��
- Linux֧�������������ϴ���vlan�ӽӿڡ�ÿ��vlan�ӽӿ����ڲ�ͬ�Ķ��������е�vlan�ӽӿ�ӵ����ͬ��MAC��ַ������Ǻ�Macvlan�ӽӿڲ�ͬ�ĵط���

## ����ģ��
![](http://172.30.40.21/blog/images/docker-macvlan/macvlan+vlan-docker.png)

## ��������
- 1.������������enp0s3�ϴ�������vlan�ӽӿڣ�
```shell
    ip link add link enp0s3 name enp0s3.200 type vlan id 200
	ip link set enp0s3.200 up
	ip link add link enp0s3 name enp0s3.201 type vlan id 201
	ip link set enp0s3.201 up
```

- 2.ʹ��docker network���������Ϊmacvlan��Docker����macvlan200��macvlan201�����ӿ�ָ����һ���贴����vlan�ӽӿڣ�
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

- 3.����macvlan200����������
```
	docker run --net=macvlan200 -idt --ip=192.168.200.2 alpine /bin/sh
	docker run --net=macvlan200 -idt --ip=192.168.200.3 alpine /bin/sh
```

- 4.����macvlan201����������
```
	docker run --net=macvlan201 -idt --ip=192.168.201.2 alpine /bin/sh
	docker run --net=macvlan201 -idt --ip=192.168.201.3 alpine /bin/sh
```

- 5.����һ�����������ظ���1��-��4�����贴������macvlan200��macvlan201������������IP�ֱ�Ϊ��192.168.200.128��192.168.200.129��192.168.201.128��192.168.201.129��


- 6.��ͬ�������ϣ���ͬvlan���������Ի�ͨ����ͬvlan���������ܻ�ͨ�������Ҫ��ͬvlan��������ͨ����Ҫ��·����������Ӧ�����á�


- 7.��������1������192.168.201.3��Ping������2������192.168.201.128��ͨ��ץ�����Կ���ICMP���д���802.1Q VLAN Tag��
![](http://172.30.40.21/blog/images/docker-macvlan/macvlan-tcpdump.png)
