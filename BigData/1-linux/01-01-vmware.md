
vmware账号：
467856641@qq.com
aA1992_1125

密钥：CC398-2YE9H-M8EQZ-ZQXEC-NURU2

# VMware网络模式

查看虚拟网络相关信息：VMware菜单-->编辑-->虚拟网络编辑器

**bridged(桥接模式)**

> 桥接模式下，宿主机物理网卡和虚拟网卡通过VMnet0虚拟交换机进行桥接，物理网卡和虚拟网卡
> 在拓扑图上处于同等地位，物理网卡和虚拟网卡处于同一个网段。虚拟交换机相当于一台现实网络中
> 的交换机，所以两个网卡的IP地址也要设置为同一网段

**Host-noly(主机模式)**

> 该模式下，虚拟系统网络是一个全封闭的网络，它为一能够访问的是宿主机，单各个虚拟机内部
> 可以互相通信。Host-Only网络和NAT网络很相似，不同的地方是Host-Only么有NAT服务，所以
> 虚拟网络不能连接到Internet。

> 宿主机和虚拟机之间的通信是通过VMware Network Adepter VMnet1虚拟网卡来实现的。同时
> 虚拟机的TCP/IP配置信息（IP地址、网关地址、DNS服务器等）都是由VMnet1虚拟网络的DHCP
> (动态主机配置协议Dynamic Host Configuration Protocol,简称DHCP。是一个局域网的网
> 络协议,该协议允许服务器向客户端动态分配 IP 地址和配置信息）服务器来动态分配的。

**NAT(网络地址转换)**

> NAT模式下，虚拟机借助NAT功能，通过宿主机器所在的网络来访问公网。在NAT网络中，
> 会使用到VMnet8虚拟交换机，宿主机上的VMare Network Adapter VMnet8虚拟网卡被
> 连接到VMnet8交换机上，来与虚拟机进行通信。但是VMare Network Adapter VMnet8
> 虚拟网卡仅仅是用于和VMnet8虚拟交换机网段通信用的，它并部位VMnet8网段提供路由功能，，
> 处于虚拟NAT网络下的虚拟机是使用虚拟的NAT服务器连接到Internet的。

> 宿主VMare Network Adapter VMnet8虚拟网卡仅仅是为Host和NAT虚拟网络下的虚拟机提供
> 一个接口，以实现虚拟机和宿主机的互访。即便卸载这块虚拟网卡，虚拟机仍然是可以上网，
> 只是宿主无法访问VMnet8网段而已。














