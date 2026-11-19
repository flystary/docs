### iproute2

- 解释

iproute2是取代网络工具（`ifconfig`、`vconfig`、`route`、`arp`等）的Linux网络工具包。大多数网络功能在`ip`命令中统一。还有用于管理流量策略（QoS）和`ss`（`netstat`替换）的`tc`

该列表只会不断增长：VRF、网络命名空间、基于策略的路由等。`ip`命令提供对所有这些功能的访问。它还提供了更统一的语法：例如，`ip link set dev eno1 nomaster`将与桥接端口、链接聚合组和VRF一起工作；

- 配置文件路径

`/etc/iproute2`中有配置文件，主要用于将符号名称分配给网络堆栈实体，如路由表。每次运行`ip`命令时，都会重新读取这些文件，

#### ip

##### 全局说明

- 缩写

```bash
ip address add 192.0.2.1/24 dev eth0
# or
ip addr a 192.0.2.1/24 dev eth0
# or
ip a a 192.0.2.1/24 dev eth0
```

- 省略

```ba
show和list单词总是可以省略的：ip address等同于ip address show和ip address list。
```

- dev不能缩写

```bash
ip a a 192.0.2.1/24 dev eth0
```

- 机器可读的输出

```bash
-o（--oneline）
用反斜杠替换每个换行符。自最早版本以来一直支持。
-br（--brief）
产生简洁的、面向机器的输出。非常适合用awk/cut进行解剖。至少从4.11开始支持。
-j（--json）
生成JSON输出，非常适合非shell脚本。支持--pretty和--brief选项。至少自4.13以来一直支持。
```

##### IPv4 or IPv6

iproute2接受虚线十进制掩码和前缀长度值。

```bash
192.0.2.10/24
192.0.2.10/255.255.255.0
```

###### 显示所有地址

```bash
ip address show #所有show命令都可以与-4或-6选项一起使用，以仅显示IPv4或IPv6地址。
```

###### 显示单个接口的地址

```shell
ip address show ${interface name}
# 示例：
ip address show eth0
```

###### 仅显示运行接口的地址

```
ip address show up
```

###### 仅显示静态或动态IPv6地址

仅显示静态配置的地址：

```
ip address show [dev ${interface}] permanent
```

###### 仅显示通过自动配置学习的地址：

```
ip address show [dev ${interface}] dynamic
```

###### 将地址添加到接口中

```
ip address add ${address}/${mask} dev ${interface name}
```

示例：

```
ip address add 192.0.2.10/27 dev eth0

ip address add 2001:db8:1::/48 dev tun10
```

您可以添加任意数量的地址。

如果您添加多个地址，您的机器将接受所有地址的数据包。您添加的第一个地址将成为“主要地址”。默认情况下，它用作传出数据包的源地址的接口的主要地址。您设置的所有附加地址都将成为次要地址。

###### 添加一个带有人类可读描述的地址

```
ip address add ${address}/${mask} dev ${interface name} label ${interface name}:${description}
```

示例：

```
ip address add 192.0.2.1/24 dev eth0 label eth0:WANaddress
```

由于一些向后兼容性问题，标签必须以接口名称开头，后跟冒号，否则您将收到错误。将标签控制在16个字符以内，否则您将收到此错误：`RTNETLINK answers: Numerical result out of range`。

备注

对于IPv6地址，此命令无效。它会正确添加地址，但会忽略标签。

###### 从界面中删除地址

```
ip address delete ${address}/${prefix} dev ${interface name}
```

示例：

```
ip address delete 192.0.2.1/24 dev eth0

ip address delete 2001:db8::1/64 dev tun1
```

需要一个接口名称——内核不会尝试自动猜测您要从哪个接口中删除该地址。这种猜测并不总是明确的：Linux确实允许在多个接口上配置相同的地址，并且它具有有效的用例（在思科世界中，这被称为“未编号接口”）。

###### 从界面中删除所有地址

```
ip address flush dev ${interface name}
```

示例：

```
ip address flush dev eth1
```

默认情况下，此命令同时删除IPv4和IPv6地址。如果您只想删除IPv4或IPv6地址，请使用`ip -4 address flush`或`ip -6 address flush`。

###### 更改主地址

无法交换主IPv4和辅助IPv4地址或显式设置新的主IPv4地址。尝试始终先设置主地址。

如果sysctl变量`net.ipv4.conf.${interface}.promote_secondaries`设置为1，当您删除主地址时，第一个次要地址将成为主地址。您可以使用`net.ipv4.conf.default.promote_secondaries=1`全局启用此行为。

请注意，当`promote_secondaries`设置为0时，删除主地址也会从其界面中删除**所有**辅助地址。此设置因Linux发行版而异，因此在尝试更改主地址之前，请仔细检查。

如果删除主地址，辅助IPv6地址始终被提升为主地址。

##### Neighbor（ARP和NDP）表

此命令支持美国（`ip neighbor`）和英国（`ip neighbour`）拼写变体。

###### 查看邻居表

```
ip neighbor show
```

所有“显示”命令都支持`-4`和`-6`选项，仅查看IPv4（ARP）或IPv6（NDP）邻居。默认情况下，所有邻居都会显示。

###### 查看指定网卡的邻居

```
ip neighbor show dev ${interface name}
```

示例：`ip neighbor show dev eth0`

###### 清除指定网卡邻居

```
ip neighbor flush dev ${interface name}
```

示例：`ip neighbor flush dev eth1`

###### 添加邻居表

```
ip neighbor add ${network address} lladdr ${link layer address} dev ${interface name}
```

示例：`ip neighbor add 192.0.2.1 lladdr 22:ce:e0:99:63:6f dev eth0`

它的一个用例是数据链路层安全的一种形式。您可以在界面上完全禁用ARP，并手动添加授权设备的MAC地址。

###### 删除邻居表

```
ip neighbor delete ${network address} lladdr ${link layer address} dev ${interface name}
```

示例：`ip neighbor delete 192.0.2.1 lladdr 22:ce:e0:99:63:6f dev eth0`

允许您删除静态条目或删除自动学习的条目，而无需刷新表。

##### 网络接口

`ip link`家族的命令执行所有接口类型通用的操

###### 显示所有网络接口信息

```bash
ip link show
ip link list
```

###### 显示指定网络接口信息

```bash
ip link show dev ${interface name}

```

```bash
# 示例：
ip link show dev eth0
ip link show dev tun10 # 省略dev词
```



###### 开启和关闭网络接口

```bash
ip link set dev ${interface name} up

ip link set dev ${interface name} down

```

```bash
# 示例：
ip link set dev eth0 down
ip link set dev br0 up
# 注意：虚拟链接（隧道、VLAN等）总是在“向下”状态下创建。你需要把它们提出来开始使用它们。
```

###### 设置网络接口别名

```bash
ip link set dev ${interface name} alias "${description}"
```

```bash
ip link set dev eth0 alias "LAN interface"

# 链接别名显示在ip link show输出中，如下：
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT qlen 1000
    link/ether 22:ce:e0:99:63:6f brd ff:ff:ff:ff:ff:ff
    alias LAN interface
```

###### 添加网络接口替代名称

内核将链接名称的长度限制为15个字符。为了克服这一限制，可以将替代名称添加到链接中。

```bash
ip link property add dev ${interface name} altname ${alternative name}
```

```bash
# 示例：
ip link property add dev eth0 altname eno1 altname enp3s0
# 替代名称可用于引用iproute命令中的链接。
$ ip link show eno1
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:11:af:63 brd ff:ff:ff:ff:ff:ff
    altname eno1
    altname enp3s0
```

###### 重命名网络接口

```bash
ip link set dev ${old interface name} name ${new interface name}
```

```bash
# 示例：
ip link set dev eth0 name lan
# 请注意，您无法重命名活动界面。在重命名它之前,需要把网络接口down
```

###### 更改网络接口MAC地址

```bash
ip link set dev ${interface name} address ${address}
```

```bash
# 以太网设备的MAC地址。要更改MAC地址，您需要像
ip link set dev eth0 address 22:ce:e0:99:63:6f
```

###### 更改网络接口MTU

```bash
ip link set dev ${interface name} mtu ${MTU value}
```

```shell
# 示例：
ip link set dev tun0 mtu 1480
MTU代表“最大传输单元”，是接口可以一次传输的帧的最大尺寸。
除了减少隧道中的碎片化外，这还用于提高支持所谓的“巨型帧”（高达9000字节的帧）的千兆以太网链路的性能。如果您的所有设备都支持千兆以太网，您可能想做类似ip link set dev eth0 mtu 9000的事情。
请注意，您可能也需要在L2交换机上配置它，其中一些默认禁用巨型帧。
```

###### 删除指定网络接口

```bash
ip link delete dev ${interface name}
# 只能删除虚拟链接，如VLAN、桥接或隧道。对于物理接口，此命令没有效果。
```

###### 启用或禁用指定接口ARP

```bash
ip link set ${interface name} arp on
ip link set ${interface name} arp off
```

###### 创建VLAN接口

```bash
ip link add name ${VLAN interface name} link ${parent interface name} type vlan id ${tag}
# 示例：
ip link add name eth0.110 link eth0 type vlan id 110
```

Linux支持的唯一VLAN类型是IEEE 802.1q VLAN；不支持ISL等遗留实现。您可以为VLAN接口使用任何名称。`eth0.110`是传统格式，但不是必需的。任何类似以太网的设备都可以是VLAN接口的父体：桥接、键合、L2隧道（GRETAP、L2TPv3...）。

###### 创建VLAN堆叠

VLAN堆叠（又名802.1ad QinQ）是一种通过另一个VLAN传输VLAN标记流量的方式。它的常见用例是这样的：假设您是服务提供商，并且您有一个客户希望使用您的网络基础设施将部分网络相互连接。他们在网络中使用多个VLAN，因此普通租赁的VLAN不是一种选择。使用QinQ，您可以在客户流量进入您的网络时向其添加第二个标签，并在退出时删除该标签，这样就不会有冲突，也无需浪费VLAN号码。

服务标签是提供商用于通过其网络传输客户端流量的VLAN标签。客户端标签是客户设置的标签。请注意，客户端VLAN接口的链接MTU不会自动调整；您需要自己处理，并将客户端接口MTU减少至少4字节，或相应地增加父MTU。符合标准的QinQ从Linux 3.10开始可用。

```bash
ip link add name ${service interface} link ${physical interface} type vlan proto 802.1ad id ${service tag}

ip link add name ${client interface} link ${service interface} type vlan proto 802.1q id ${client tag}
```

```bash
# 示例
ip link add name eth0.100 link eth0 type vlan proto 802.1ad id 100 # Create a service tag interface
ip link add name eth0.100.200 link eth0.100 type vlan proto 802.1q id 200 # Create a client tag interface
```

###### 创建虚拟MAC（MACVLAN）接口

```bash
ip link add name ${macvlan interface name} link ${parent interface} type macvlan
# 示例：
ip link add name peth0 link eth0 type macvlan
#MACVLAN接口就像单个接口上的“辅助MAC地址”。从用户的角度来看，它们看起来像普通的以太网接口，并处理其MAC地址的所有流量。
```

###### 创建虚拟接口

```bash
ip link add name ${dummy interface name} type dummy
# 示例：
ip link add name dummy0 type dummy
```

###### 创建桥接接口

桥接接口是虚拟以太网交换机。您可以使用它们将Linux盒子变成缓慢的L2交换机，或启用虚拟机管理程序主机上的虚拟机之间的通信

```bash
ip link add name ${bridge name} type bridge
# 示例：
ip link add name br0 type bridge
```

###### 添加桥接端口

添加到桥接器的接口成为虚拟交换机端口。它仅在数据链路层上运行，并停止所有网络层操作。

```bash
ip link set dev ${interface name} master ${bridge name}
# 示例：
ip link set dev eth0 master br0
```

###### 删除桥接端口

```bash
ip link set dev ${interface name} nomaster
# 示例：
ip link set dev eth0 nomaster
```

###### 创建接口聚合（bond）接口

```bash
ip link add name ${name} type bond
# 示例：
ip link add name bond1 type bond
```

###### 创建中间功能块接口

中间功能块设备与tc一起用于流量重定向和镜像。

```bash
ip link add ${interface name} type ifb
# 示例：
ip link add ifb10 type ifb
```

###### 创建一对虚拟以太网接口

虚拟以太网（veth）设备总是成对出现，并作为双向管道工作：任何进入其中一个设备的东西都会从另一个设备中出来。它们与网络命名空间和容器（OpenVZ或LXC）等系统分区功能结合使用，用于将一个分区连接到另一个分区。

```bash
ip link add name ${first device name} type veth peer name ${second device name}
# 示例：
ip link add name veth-host type veth peer name veth-guest
```

这些设备可能以向下或向上状态创建。在最新版本（在5.10中测试）中，它们已下降。最好假设他们已经关闭，并始终使用`ip link set ${intf} up`。

##### 接口组

链路组类似于托管交换机中的端口范围。您可以将网络接口添加到一个编号组，并同时对该组的所有接口执行操作。未分配给任何组的链接属于0组（“默认”）。

###### 向组内添加接口

```bash
ip link set dev ${interface name} group ${group number}
# 示例：
ip link set dev eth0 group 42
ip link set dev eth1 group 42
```

###### 从组内删除接口

这可以通过将其分配给默认组来完成。

```bash
ip link set dev ${interface name} group 0
ip link set dev ${interface} group default
# 示例：
ip link set dev tun10 group 0
```

###### 为组分配别名

组名在`/etc/iproute2/group`文件中配置。组0的符号名称“默认”不是iproute2内置的；它来自默认组配置文件。您可以按照相同的`${number} ${name}`格式添加自己的，每行一个。您最多可以有255个命名组。

一旦您配置了组名，其编号和名称可以在`ip`命令中互换使用。

示例：

```bash
echo "10    customer-vlans" >> /etc/iproute2/group
```

之后，您可以在所有操作中使用该名称，例如：

```bash
ip link set dev eth0.100 group customer-vlans
```

###### 对组进行操作

```bash
ip link set group ${group number} ${operation and arguments}
```

示例：

```bash
ip link set group 42 down
ip link set group uplinks mtu 1200
```

###### 查看特定组的的信息

使用带有`group ${group}`修饰符的常规信息查看命令。

示例：

```bash
ip link list group 42
ip address show group customers
```

##### TUN和TAP设备

TUN和TAP设备允许用户空间程序模拟网络设备。两者之间的区别在于，TAP设备与以太网帧（L2设备）一起工作，而TUN与IP数据包（L3设备）一起工作。TUN/TAP设备有两种类型：持久和瞬态。瞬态TUN/TAP设备由用户空间程序在打开特殊设备时创建，并在关联的文件描述符关闭时自动销毁。

###### 查看TUN/TAP设备

```bash
ip tuntap show

# 缩写
ip tuntap
```

此命令是确定某些设备是否处于TUN或TAP模式的唯一方法。

###### 添加管理员用户使用的TUN/TAP设备

```bash
ip tuntap add dev ${interface name} mode ${mode}
# 示例：
ip tuntap add dev tun0 mode tun
ip tuntap add dev tap9 mode tap
```

###### 添加普通用户使用的TUN/TAP设备

```bash
ip tuntap add dev ${interface name} mode ${mode} user ${user} group ${group}
# 示例：
ip tuntap add dev tun1 mode tun user me group mygroup
ip tuntap add dev tun2 mode tun user 1000 group 1001
```

###### 使用备用数据包格式添加TUN/TAP设备

将元信息添加到通过文件描述符接收的每个数据包中。很少有程序期望此信息，而不期望此信息的程序在此模式下的设备无法正常运行。

```bash
ip tuntap add dev ${interface name} mode ${mode} pi
# 示例：
ip tuntap add dev tun1 mode tun pi
```

###### 添加TUN/TAP忽略流量控制

通常，发送到TUN/TAP设备的数据包与发送到任何其他设备的数据包的传输方式相同：它们被放入由交通控制引擎（由`tc`命令配置）处理的队列中。这可以绕过，从而禁用此TUN/TAP设备的交通控制引擎。

```bash
ip tuntap add dev ${interface name} mode ${mode} one_queue
# 示例：
ip tuntap add dev tun1 mode tun one_queue
```

###### 删除 TUN/TAP 设备

```bash
ip tuntap delete dev ${interface name} mode ${mode}
```

```bash
# 示例：
ip tuntap delete dev tun0 mode tun
ip tuntap delete dev tap1 mode tap
```

注意：您必须指定模式。该模式不会显示在`ip link show`中，因此，如果您不知道它是TUN还是TAP设备，请咨询`ip tuntap show`的输出。

##### 隧道接口

将整个数据包封装到另一个协议中来模拟路由网络上的直接连接。

Linux目前支持IPIP（IPv4中的IPv4）、SIT（IPv4中的IPv6）、IP6IP6（IPv6中的IPv6）、IPIP6（IPv6中的IPv4）、GRE（几乎任何东西）和VTI（IPsec中的IPv4）。

请注意，隧道是在DOWN状态下创建的；您需要调出它们。

在本节中`${local endpoint address}`和`${remote endpoint address}`是指分配给物理接口的地址，而`${address}`是指分配给隧道接口的地址。

###### 创建IPIP隧道

```bash
ip tunnel add ${interface name} mode ipip local ${local endpoint address} remote ${remote endpoint address}

# 示例：
ip tunnel add tun0 mode ipip local 192.0.2.1 remote 198.51.100.3
ip link set dev tun0 up
ip address add 10.0.0.1/30 dev tun0
```

###### 创建SIT（6in4）隧道

这种类型的隧道通常用于提供具有IPv6连接的IPv4连接网络。有所谓的“隧道经纪人”向所有感兴趣的人提供它，例如Hurricane Electric的tunnebroker.net。

```bash
ip tunnel add ${interface name} mode sit local ${local endpoint address} remote ${remote endpoint address}
# 示例：
ip tunnel add tun9 mode sit local 192.0.2.1 remote 198.51.100.3
ip link set dev tun9 up
ip address add 2001:db8:1::1/64 dev tun9
```

###### 创建一个IPIP6隧道

```bash
ip -6 tunnel add ${interface name} mode ipip6 local ${local endpoint address} remote ${remote endpoint address}
# 示例：
ip -6 tunnel add tun8 mode ipip6 local 2001:db8:1::1 remote 2001:db8:1::2
```

###### 创建IP6IP6隧道

```bash
ip -6 tunnel add ${interface name} mode ip6ip6 local ${local endpoint address} remote ${remote endpoint address}
# 示例：
ip -6 tunnel add tun3 mode ip6ip6 local 2001:db8:1::1 remote 2001:db8:1::2
ip link set dev tun3 up
ip address add 2001:db8:2:2::1/64 dev tun3

```

###### 创建L2 GRE隧道设备

静态GRE隧道传统上用于封装IPv4或IPv6数据包，但[RFC](https://tools.ietf.org/html/rfc2784)不会将GRE有效负载限制为L3协议数据包。可以封装任何东西，包括以太网帧。然而，在Linux中，`gre`封装特指L3设备，而对于能够传输以太网帧的L2设备，您需要使用`gretap`封装。

```bash
# L2 GRE over IPv4
ip link add ${interface name} type gretap local ${local endpoint address} remote ${remote endpoint address}

# L2 GRE over IPv6
ip link add ${interface name} type ip6gretap local ${local endpoint address} remote ${remote endpoint address}
```

这些隧道可以与其他物理和虚拟接口桥接。

```bash
# 示例
ip link add gretap0 type gretap local 192.0.2.1 remote 203.0.113.3
ip link add gretap1 type ip6gretap local 2001:db8:dead::1 remote 2001:db8:beef::2
```

###### GRE隧道

```bash
ip tunnel add ${interface name} mode gre local ${local endpoint address} remote ${remote endpoint address}
# 示例：
ip tunnel add tun6 mode gre local 192.0.2.1 remote 203.0.113.3
ip link set dev tun6 up
ip address add 192.168.0.1/30 dev tun6
ip address add 2001:db8:1::1/64 dev tun6
```

GRE可以同时封装IPv4和IPv6。然而，默认情况下，它使用IPv4进行传输，对于IPv6上的GRE，有一个单独的隧道模式，ip6gre。

###### 创建多对一GRE隧道

它只是一个标识符，允许路由器区分一个隧道和另一个隧道，因此您可以在相同的端点之间创建多个隧道。

```bash
ip tunnel add ${interface name} mode gre local ${local endpoint address} remote ${remote endpoint address} key ${key value}

# 示例：
ip tunnel add tun4 mode gre local 192.0.2.1 remote 203.0.113.6 key 123
ip tunnel add tun5 mode gre local 192.0.2.1 remote 203.0.113.6 key 124

```

您还可以以点十进制IPv4格式指定密钥。

###### 创建一对多GRE隧道

```bash
ip tunnel add ${interface name} mode gre local ${local endpoint address} key ${key value}

# 示例：
ip tunnel add tun8 mode gre local 192.0.2.1 key 1234
ip link set dev tun8 up
ip address add 10.0.0.1/27 dev tun8
```

在没有远程端点地址的情况下，密钥是识别隧道流量的唯一方法，因此需要`${key value}`。

这种类型的隧道允许您使用相同的隧道接口与多个端点进行通信。它通常用于具有多个端点相互通信的复杂VPN设置（在思科术语中，“动态多点VPN”）。

由于没有明确的远程端点地址，因此仅仅创建一个隧道显然是不够的。您的系统需要知道其他端点在哪里。在现实生活中，NHRP（下一跳分辨率协议）被用于它。

对于测试，您可以手动添加对等体（给定远程端点在其物理接口上使用203.0.113.6地址，在隧道上使用10.0.0.2地址）：

```bash
ip neighbor add 10.0.0.2 lladdr 203.0.113.6 dev tun8
```

您也必须在远程端点上执行此操作：

```bash
ip neighbor add 10.0.0.1 lladdr 192.0.2.1 dev tun8
```

###### 在IPv6上创建GRE隧道

```bash
ip -6 tunnel add name ${interface name} mode ip6gre local ${local endpoint} remote ${remote endpoint}
```

###### 删除隧道

```bash
ip tunnel del ${interface name}
# 示例：
ip tunnel del gre1
```

请注意，在较旧的iproute2版本中，此命令不支持完整的`delete`语法，只支持`del`。最新版本允许完整和缩写形式（在iproute2-ss131122中测试）。

###### 修改隧道

```bash
ip tunnel change ${interface name} ${options}
# 示例：
ip tunnel change tun0 remote 203.0.113.89
ip tunnel change tun10 key 23456
```

注意：显然，您无法将密钥添加到之前未键控的隧道中。不确定这是一个错误还是一个功能。此外，由于明显的原因，您无法即时更改隧道模式。

###### 查看隧道

```bash
ip tunnel show
ip tunnel show ${interface name}

# 示例：
$ ip tun show tun99
tun99: gre/ip  remote 10.46.1.20  local 10.91.19.110  ttl inherit
```

##### L2TPv3

[L2TPv3](https://tools.ietf.org/html/rfc3931)是一种通常用于L2伪线的隧道协议。

在许多发行版本中，L2TPv3是作为模块编译的，默认情况下可能不会加载。如果运行任何`ip l2tp`命令产生错误，如`RTNETLINK answers: No such file or directory`，并且`Error talking to the kernel`，您需要加载`l2tp_netlink`和`l2tp_eth`内核模块。如果您想在IP上而不是UDP上使用L2TPv3，也请加载`l2tp_ip`。

与Linux中的其他隧道协议实现相比，L2TPv3术语有些落后。您创建一个*隧道*，然后将*会话*绑定到它。您可以将具有不同标识符的多个会话绑定到同一个隧道。虚拟网络接口（默认情况下称为`l2tpethX`）与会*话*而不是*隧道*相关联。

您只能使用iproute2创建*静态*（非托管）L2TPv3隧道。如果您想将L2TP用于远程访问VPN，或者需要动态创建的伪线，您需要一个用户空间守护进程来处理它。这超出了本文档的范围。

###### 在UDP上创建L2TPv3隧道

```bash
ip l2tp add tunnel \
tunnel_id ${local tunnel numeric identifier} \
peer_tunnel_id ${remote tunnel numeric identifier} \
udp_sport ${source port} \
udp_dport ${destination port} \
encap udp \
local ${local endpoint address} \
remote ${remote endpoint address}
```

示例：

```bash
ip l2tp add tunnel \
tunnel_id 1 \
peer_tunnel_id 1 \
udp_sport 5000 \
udp_dport 5000 \
encap udp \
local 192.0.2.1 \
remote 203.0.113.2
```

注意：两个端点上的隧道标识符和其他设置必须匹配。

###### 通过IP创建一个L2TPv3隧道

```bash
ip l2tp add tunnel \
tunnel_id ${local tunnel numeric identifier} \
peer_tunnel_id {remote tunnel numeric identifier } \
encap ip \
local 192.0.2.1 \
remote 203.0.113.2
```

在IP而不是UDP中封装L2TPv3帧会减少开销，但可能会给NAT后面的端点带来问题。

###### 创建一个L2TPv3会话

```bash
ip l2tp add session tunnel_id ${local tunnel identifier} \
session_id ${local session numeric identifier} \
peer_session_id ${remote session numeric identifier}
```

```bash
# 示例：
ip l2tp add session tunnel_id 1 \
session_id 10 \
peer_session_id 10
```

注意：`tunnel_id`值必须与现有隧道的值匹配（如果不存在，iproute2不会创建隧道）。两个端点上的会话标识符必须匹配。

一旦您创建隧道和会话，`l2tpethX`接口将以DOWN状态出现。将状态更改为[UP](https://baturin.org/docs/iproute2/#ip-link-set-up-down)，并将其与另一个接口桥接或为其分配地址。

###### 删除 L2TPv3 会话

```bash
ip l2tp del session tunnel_id ${tunnel identifier} \
session_id ${session identifier}
```

示例：`ip l2tp del session tunnel_id 1 session_id 1`

###### 删除L2TPv3隧道

```bash
ip l2tp del tunnel tunnel_id ${tunnel identifier}
# 示例：
ip l2tp del tunnel tunnel_id 1
```

注意：在删除隧道本身之前，您需要删除与隧道关联的所有会话。

###### 查看L2TPv3隧道信息

```bash
ip l2tp show tunnel
ip l2tp show tunnel tunnel_id ${tunnel identifier}
# 示例：
ip l2tp show tunnel tunnel_id 12
```

###### 查看L2TPv3会话信息

```bash
ip l2tp show session
ip l2tp show session session_id ${session identifier} \
tunnel_id ${tunnel identifier}
# 示例：
ip l2tp show session session_id 1 tunnel_id 12
```

##### VXLAN

VXLAN是一种专为分布式交换网络设计的隧道协议。它通常用于虚拟化设置，以将虚拟网络拓扑与底层物理网络的拓扑解耦。

VXLAN可以在多播或单播模式下工作，并支持使用VNI（虚拟网络标识符）隔离虚拟网络，类似于以太网网络中的VLAN。

多播模式的缺点是，您需要使用多播路由协议，通常是PIM-SM，以使其在路由网络上工作，但如果您设置了它，则无需手动创建所有VXLAN连接。

VXLAN的底层封装协议是UDP。

###### 创建单播VXLAN链接

```bash
ip link add name ${interface name} type vxlan \
   id <0-16777215> \
   dev ${source interface}\
   remote ${remote endpoint address} \
   local ${local endpoint address} \
   dstport ${VXLAN destination port}
```

示例：

```bash
ip link add name vxlan0 type vxlan \
   id 42 dev eth0 remote 203.0.113.6 local 192.0.2.1 dstport 4789
```

注意：`id`选项是VXLAN网络标识符（VNI）。

###### 创建多播VXLAN链接

```bash
ip link add name ${interface name} type vxlan \
   id <0-16777215> \
   dev ${source interface} \
   group ${multicast address} \
   dstport ${VXLAN destination port}
```

示例：

```bash
ip link add name vxlan0 type vxlan \
   id 42 dev eth0 group 239.0.0.1 dstport 4789
```

之后，您需要并与另一个接口进行桥接或为其分配地址。

##### 路由表

对于IPv4路由，您可以使用前缀长度或虚线十进制子网掩码。也就是说，192.0.2.0/24和192.0.2.0/255.255.255.0都同样可以接受。

Linux内核不会保留无法到达的下一跳的路由。如果链接下降，所有使用该链接的路由都会从路由表中永久删除。您可能没有注意到这种行为，因为在许多情况下，当链接上下时，额外的软件（例如NetworkManager或rp-pppoe）会负责恢复路由。

###### 查看所有路由

```bash
ip route
ip route show
```

您可以使用`-4`和`-6`选项仅查看IPv4或IPv6路由。默认情况下，仅显示IPv4路由。要查看IPv6路由，请使用`ip -6 route`。

###### 查看网络及其所有子网的路由

```bash
ip route show to root ${address}/${mask}
```

例如，如果您在网络中使用192.168.0.0/24子网，并且它被分为192.168.0.0/25和192.168.0.128/25，则可以看到所有带有`ip route show to root 192.168.0.0/24`的路由。

注意：“to”一词在所有“show”命令中都是可选的。

###### 查看网络和所有超网的路由

```bash
ip route show to match ${address}/${mask}
```

如果您想查看到192.168.0.0/24的路由和所有较大的子网，请使用`ip route show to match 192.168.0.0/24`。

路由器更喜欢更具体的路由而不是不太具体的路由，因此，当特定子网的流量以错误方式发送时，这通常有助于调试，因为缺少通往该子网的路由，但存在通往更大子网的路由。

###### 查看到确切子网的路线

```shell
ip route show to exact ${address}/${mask}
```

如果您想查看192.168.0.0/24的路线，但不想查看，例如192.168.0.0/25和192.168.0.0/16，您可以使用`ip route show to exact 192.168.0.0/24`。

###### 只查看内核实际使用的路线

```bash
ip route get ${address}/${mask}
```

示例：`ip route get 192.168.0.0/24`。

请注意，此命令总是返回一条路线。在大多数情况下，这不是问题，但在多路径路由设置中，结果可能无法反映完整画面，因此也不要忘记查看相应的“显示”命令输出。

###### 查看路由缓存（仅限前3.6内核）

```bash
ip route show cached
```

在3.6版本之前，Linux使用路由缓存。在旧内核中，此命令显示路由缓存的内容。它可以与上面描述的修饰符一起使用。在较新的内核中，它无所作为。

###### 通过网关添加路由

```bash
ip route add ${address}/${mask} via ${next hop}
```

示例：

```bash
ip route add 192.0.2.128/25 via 192.0.2.1

ip route add 2001:db8:1::/48 via 2001:db8:1::1
```

###### 通过网络接口添加路由

```bash
ip route add ${address}/${mask} dev ${interface name}
# 示例：
ip route add 192.0.2.0/25 dev ppp0
```

接口路由通常与PPP隧道等点对点接口一起使用。由于没有可能将多个主机连接到点对点接口，因此在这些情况下也无需明确指定网关地址。

###### 添加一条没有一致性检查网关可访问性的路线

```bash
ip route add ${address}/${mask} dev ${interface name} onlink
# 示例：
ip route add 192.0.2.128/25 via 203.0.113.1 dev eth0 onlink
```

`onlink`关键字禁用内核的网关一致性检查，并允许通过看起来无法访问的网关添加路由。在隧道配置和容器/虚拟化网络中非常有用，其中同一链路上的多个网络使用单个网关。

###### 更改或替换路线

对于修改路由，有`ip route change`和`ip route replace`命令。它们之间的区别在于，如果您尝试更改不存在的路由，`change`命令将产生错误。`replace`命令将创建一个路由，如果它还不存在。



```bash
# 示例：
ip route change 192.168.2.0/24 via 10.0.0.1
ip route replace 192.0.2.1/27 dev tun0
```

###### 删除一条路由

```bash
ip route delete ${route specifier}
```



```bash
# 示例：
ip route delete 10.0.1.0/25 via 10.0.0.1
ip route delete default dev ppp0
```

###### 默认路线

有一个创建默认路由的快捷方式。

```bash
ip route add default via ${address}/${mask}

ip route add default dev ${interface name}
```

这些相当于：

```bash
ip route add 0.0.0.0/0 via ${address}/${mask}

ip route add 0.0.0.0/0 dev ${interface name}
```

对于IPv6路由，`default`等同于::/0。

```bash
ip -6 route add default via 2001:db8::1
```

###### 黑洞路由

```bash
ip route add blackhole ${address}/${mask}
# 示例：
ip route add blackhole 192.0.2.1/32。
```

前往与黑洞路线匹配的目的地的流量被默默丢弃。

黑洞路由有两个用例。首先，它们可以作为非常快速的出站流量过滤器，例如，使已知的僵尸网络控制器无法访问，或保护网络内的服务器免受传入的DDoS攻击。其次，它们可以用来欺骗路由协议守护进程，让其认为如果您只有通往网络部分的真实路由，但又想将其聚合起来，则认为您有通往网络的路由。

###### 其他特殊路由

还有其他几种特殊用途的路由。

```bash
ip route add unreachable ${address}/${mask}
ip route add prohibit ${address}/${mask}
ip route add throw ${address}/${mask}
```

这些路由使系统丢弃数据包，并向发件人回复ICMP错误消息。

- `unreachable`：发送ICMP“无法联系到主机”。
- `prohibit`：发送ICMP“行政禁止”。
- `throw`：发送“net无法到达”。

与黑洞路由不同，这些不能推荐用于阻止不需要的流量（例如DDoS），因为它们为每个丢弃的数据包生成一个回复数据包，从而创造更大的流量。它们可能适合实施内部访问策略，但防火墙通常是一个更好的主意。

“投掷”路由可用于实现基于策略的路由。在非默认的表格中，它们会停止查找过程，但不会发送ICMP错误消息。

###### 具有不同指标的路线

```bash
ip route add ${address}/${mask} via ${gateway} metric ${number}
```

示例：

```bash
ip route add 192.168.2.0/24 via 10.0.1.1 metric 5
ip route add 192.168.2.0 dev ppp0 metric 10
```

如果同一网络有多个路由具有不同的度量值，内核更喜欢度量*最低*的路由。

这个概念的一个重要部分是，当接口崩溃时，因此事件而变得无用的路由从路由表中消失，系统将回退到具有更高度量值的路由。此功能通常用于实现到重要目的地的备份连接。

###### 多路径路由

```bash
ip route add ${addresss}/${mask} nexthop via ${gateway 1} weight ${number} nexthop via ${gateway 2} weight ${number}
```

多路径路由使系统根据权重跨多个链路平衡数据包（首选权重较高，因此权重为2的网关/接口的流量大约是权重1的网关/接口的两倍）。您可以拥有任意数量的网关，并且可以混合网关和接口路由：

```bash
ip route add default nexthop via 192.168.1.1 weight 1 nexthop dev ppp0 weight 10
```

警告：这种类型的负载平衡的缺点是，数据包不能保证通过它们进入的同一链接发回。这被称为“不对称路由”。对于只是转发数据包而不进行任何本地流量处理的路由器来说，这通常很好，在某些情况下甚至是不可避免的。

如果您的系统进行一些本地处理（例如NAT），这可能会导致传入连接出现问题。在这种情况下，您应该使用有状态的L4负载平衡设置。

##### 基于策略的路由

Linux中基于策略的路由（PBR）设计如下：首先创建自定义路由表，然后创建规则，告诉内核表用于哪些数据包。

一些表是预定义的：

```bash
local（表255）：包含到本地和广播地址的控制路由。
main（表254）：包含所有非PBR路由。如果您在添加路由时没有指定表，它将转到此表。
default（表253）：保留用于后处理，通常未使用。
当您向它们添加第一条路由时，用户定义的表会自动创建。
```



###### 创建策略路线

```bash
ip route add ${route options} table ${table id or name}
# 示例：
ip route add 192.0.2.0/27 via 203.0.113.1 table 10
ip route add 0.0.0.0/0 via 192.168.0.1 table ISP2
ip route add 2001:db8::/48 dev eth1 table 100
```

注意：您也可以使用路由表部分中描述的任何路由选项进行策略路由，唯一的区别是末尾的table ${table id/name}部分。

数字表标识符和名称可以互换使用。要创建您自己的符号名称，请编辑/etc/iproute2/rt_tables配置文件。

delete、change、replace和所有其他路由操作也适用于任何表。

ip route ... table main或ip route ... table 254与没有表部分的命令具有完全相同的效果。

###### 查看策略路由

```bash
ip route show table ${table id or name}
# 示例：
ip route show table 100
ip route show table test
```

注意：在这种情况下，您需要show词；像ip route table 120这样的速记不起作用，因为命令是模棱两可的。

###### 一般规则语法

```bash
ip rule add ${options} <lookup ${table id or name}|blackhole|prohibit|unreachable>
```

如果使用lookup操作，匹配${options}（如下所述）的流量将根据具有指定名称/id的表而不是“main”/254表进行路由。

blackhole、prohibit和unreachable的操作就像在默认表中一样。

对于IPv6规则，使用ip -6，其余的语法是一样的。

table ${table id or name}可以用作lookup ${table id or name}的快捷方式。

###### 创建匹配源网络的规则

```bash
ip rule add from ${source network} ${action}
# 示例：
ip rule add from 192.0.2.0/24 lookup 10
ip -6 rule add from 2001:db8::/32 prohibit
```

注意：all关键字都可以用作0.0.0.0/0或::/0的快捷方式

###### 创建一个规则来匹配目标网络

```bash
ip rule add to ${destination network} ${action}
# 示例：
ip rule add to 192.0.2.0/24 blackhole
ip -6 rule add to 2001:db8::/32 lookup 100
```

###### 创建一个规则来匹配ToS字段值

```bash
ip rule add tos ${ToS value} ${action}
# 示例：
ip rule add tos 0x10 lookup 110。
```

###### 创建匹配防火墙标记值的规则

```bash
ip rule add fwmark ${mark} ${action}
# 示例：
ip rule add fwmark 0x11 lookup 100。
```

确保在路由决定之前处理的防火墙链中设置标记，否则使用该标记的PBR规则将不起作用。您可以在Phil Hagen的博客中找到一个优秀的netfilter流程图。对于转发流量，mangle FORWARD应该是一个好地方，例如iptables -t mangle -I FORWARD -s 192.0.2.1 -j MARK --set-mark 0x11。

###### 创建一个规则来匹配入站接口

```bash
ip rule add iif ${interface name} ${action}
# 示例：
ip rule add iif eth0 lookup 10
ip rule add iif lo lookup 20
```

带有iif lo（环回）的规则将匹配本地生成的流量。

###### 创建一个规则来匹配出站接口

```
ip rule add oif ${interface name} ${action}
示例：ip rule add oif eth0 lookup 10。
```

注意：这仅适用于本地生成的流量。

###### 创建匹配用 户ID范围的规则

```bash
ip rule add uidrange <${start}-${end}>
# 示例：
ip rule add uidrange 1000-1100 lookup 10
ip rule add uidrange 3000-3000 lookup 20
```

要将规则应用于单个用户，请对范围的开始和结束使用相同的UID。

###### 设置规则优先级

```bash
ip rule add ${options} ${action} priority ${value}
# 示例：
ip rule add from 192.0.2.0/25 lookup 10 priority 10
ip rule add from 192.0.2.0/24 lookup 20 priority 20
```

注意：规则从最低优先级到最高优先级遍历，并在第一场比赛中停止处理，因此您需要将更具体的规则放在不太具体的规则之前。上面的示例展示了192.0.2.0/24及其子网192.0.2.0/25的规则。如果优先级被颠倒，并将/25的规则放在/24的规则之后，就永远无法达到。

###### 显示所有规则

```bash
ip rule show
ip -6 rule show
```

###### 删除规则

```bash
ip rule del ${options} ${action}
# 示例：
ip rule del 192.0.2.0/24 lookup 10
```

注意：您可以从ip rule show或ip -6 rule show的输出中复制/粘贴。

###### 删除所有规则

```bash
ip rule flush
ip -6 rule flush
```

注意：此操作具有高度破坏性。即使您没有配置任何规则，默认情况下也会为您创建一些基本规则，例如from all lookup main规则。在未配置的机器上，你可以看到以下内容：

```bash
$ ip rule show
0:	from all lookup local
32766:	from all lookup main
32767:	from all lookup default
```

```bash
$ ip -6 rule show
0:	from all lookup local
32766:	from all lookup main
```

The from all lookup local rule is special and cannot be deleted. The from all lookup main is not, there may be valid reasons not to have it, e.g., if you want to route only traffic you created explicit rules for. As a side effect, if you do ip rule flush, that rule will be deleted, which will make the system stop routing any traffic until you restore your rules.

##### VRF

VRF（虚拟路由和转发）是一种在单独的路由表中隔离网络路由的机制。它允许具有冲突地址范围的多个网络在同一路由器中共存。它最常见的用例是多租户设置和提供商支持的VPN，客户可以使用自己的网络地址。

与基于策略的路由的主要区别是，PBR中使用的“正常”非默认表只分隔静态路由，而不是连接路由，因此它们无法解决地址冲突。当网络接口绑定到VRF时，来自它的所有连接的路由都将移动到一个单独的路由表。

与网络命名空间不同，VRF仅在网络层上工作。它们不会创建网络堆栈的单独副本，不会干扰LLDP等L2协议，一个进程可以将套接字绑定到多个VRF（对动态路由协议或IPsec守护程序有用）。

###### 创建一个VRF

```bash
ip link add ${name} type vrf table ${table}
# 示例：
sudo ip link add foo type vrf table 100
```

如果路由表100不存在，它将自动创建。

###### 查看已配置的VRF

```bash
ip vrf show
ip link show vrf ${vrf}
```

ip vrf show的输出仅显示VRF和路由表之间的关联，但不显示VRF和网络接口之间的关联。要查看VRF的所有接口，请使用ip link show vrf ${vrf}。

###### 将接口绑定到VRF

```bash
ip link set ${interface} master ${vrf}
# 示例：
ip link set eth0 master foo。
```

与接口相关的所有连接路由都将移动到VRF表中。例如，如果eth0具有地址192.0.2.1/24，并且VRF实例foo使用表100，则到192.0.0/24的路由将从主表中消失，并重新出现在表100中。

###### 从VRF中删除接口

```bash
ip link set ${interface} nomaster
```

与该接口关联的连接路由将移回主表。

###### 在VRF中运行命令

```bash
ip vrf exec ${vrf} ${command}
# 示例：
ip vrf exec ping 192.0.2.100。
```

该过程的流量将根据VRF表路由进行路由，因此对故障排除很有用。

##### 网络命名空间

网络命名空间是单台机器中的孤立网络堆栈实例。它们可用于安全域分离、管理虚拟机之间的流量流等。

每个命名空间都是网络堆栈的完整副本，具有自己的接口、地址、路由等。您可以在命名空间内运行进程，并将命名空间桥接到物理接口。

###### 创建一个命名空间

```bash
ip netns add ${namespace name}
# 示例：
ip netns add foo
```

###### 列出现有的命名空间

```bash
ip netns list
```

###### 删除一个命名空间

```bash
ip netns delete ${namespace name}
# 示例：
ip netns delete foo。
```



###### 在命名空间中运行进程

```bash
ip netns exec ${namespace name} ${command}
# 示例：
ip netns exec foo /bin/sh
```

注意：将进程分配给非默认命名空间需要根权限。

您可以在命名空间中运行任何进程，特别是，您可以运行/sbin/ip本身。本节中像ip netns exec foo ip link list这样的命令并不特别：我们只是在命名空间中执行另一个ip副本。您也可以在命名空间中运行交互式shell。

###### 列出分配给命名空间的所有进程

```bash
ip netns pids ${namespace name}
```

输出将是PID列表。

###### 识别进程的主要命名空间

```bash
ip netns identify ${pid}
示例：ip netns identify 9000。
```



###### 将网络接口分配给命名空间

```bash
ip link set dev ${interface name} netns ${namespace name}
ip link set dev ${interface name} netns ${pid}

# 示例：
ip link set dev eth0.100 netns foo
```

注意：一旦您将接口分配给命名空间，它将从默认命名空间中消失，您必须通过ip netns exec ${netspace name}执行所有操作，如ip netns exec ${netspace name} ip link set dev dummy0 down。

此外，当您将接口移动到另一个命名空间时，它会丢失所有现有配置，例如在其上配置的IP地址，并进入DOWN状态。您需要将其带回并重新配置。

如果您指定PID而不是命名空间名称，则接口将分配给具有该PID的进程的主要命名空间。通过这种方式，您可以将接口重新分配回默认命名空间，例如，

```bash
ip netns exec ${namespace name} ip link set dev ${intf} netns 1 #（因为init或其他具有PID 1的进程几乎可以保证在默认命名空间中）。
```

###### 将一个命名空间连接到另一个命名空间

这可以通过创建一对veth链接并将其分配给不同的命名空间来完成。

假设您想将名为“foo”的命名空间连接到默认命名空间。首先，创建一对 veth 设备：

```bash
ip link add name veth1 type veth peer name veth2
```

将 veth2 移动到命名空间 foo：

```bash
ip link set dev veth2 netns foo。
```

带上veth2，并在“foo”命名空间中添加一个地址：

```bash
ip netns exec foo ip link set dev veth2 up
```

ip netns exec foo ip address add 10.1.1.1/24 dev veth2
向veth1添加一个地址，该地址保留在默认命名空间中：

```bash
ip address add 10.1.1.2/24 dev veth1
```

现在，您可以ping 10.1.1.1，这位于foo命名空间中，并设置在该命名空间的其他接口中配置的子网的路由。

如果您想要切换而不是路由，您可以将这些veth接口与相应命名空间中的其他接口桥接。相同的技术可用于将命名空间连接到物理网络。

##### 多播和路由

多播主要由应用程序和路由守护程序处理，因此您在这里可以也应该手动完成的不多。与多播相关的`ip`命令对调试最有用。

###### 查看多播组

```bash
ip maddress show
ip maddress show ${interface name}
```

示例：

```bash
$ ip maddress show dev lo
1:	lo
	inet  224.0.0.1
	inet6 ff02::1
	inet6 ff01::1
```

###### 添加链接层多播地址

您无法手动加入IP多播组，但您可以添加多播MAC地址（即使很少需要）。

```bash
ip maddress add ${MAC address} dev ${interface name}
# 示例：
ip maddress add 01:00:5e:00:00:ab dev eth0
```

###### 查看多播路线

多播路由无法手动添加，因此此命令只能显示路由守护进程安装的多播路由。它支持与单播路由查看命令（`iif`、`table`、`from`等）相同的修饰符。

```bash
ip mroute show
```

##### 网络事件监控

您可以使用iproute2监控某些网络事件，例如网络配置、路由表和ARP/NDP表的更改。

###### 监控所有事件

您可以在没有参数的情况下调用命令，也可以显式指定`all`。

```bash
ip monitor
ip monitor all
```

###### 监控特定事件

```bash
ip monitor ${event type}
```

事件类型可以是：

- 接口

  接口状态：上下下的接口，创建或销毁的虚拟接口等。

- IP地址

  链接地址更改。

- 路由

  路由表更改。

- mroute

  多播路由更改。

- 邻居

  邻居（ARP和NDP）表的变化。

当有不同的IPv4和IPv6子系统时，通常的`-4`和`-6`选项允许您仅显示指定协议的事件。

```bash
ip -4 monitor route
ip -6 monitor neigh
ip -4 monitor address
```

###### 阅读由rtmon生成的日志文件

iproute2包括一个名为`rtmon`的程序，其目的基本相同，但将事件写入二进制日志文件，而不是显示它们。您可以使用`ip monitor`命令读取这些日志文件。

```bash
ip monitor ${event type} file ${path to the log file}
```

rtmon语法与`ip monitor`的语法相似，只是事件类型仅限于`link`、`address`、`route``all`；地址系列在`-family`选项中指定。

```bash
rtmon [-family <inet|inet6>] [<route|link|address|all>] file ${log file path}
```

##### netconf

###### 查看所有接口的sysctl配置

```bash
ip netconf show
```

###### 查看特定接口的sysctl配置

```bash
ip netconf show dev ${interface}
# 示例：
ip netconf show dev eth0
```



























#### tc

tc 则是属于 iproute2 中的 流量控制 部分，正如 tc 的全称 “traffic control” 。

```bash
                Userspace programs
                     ^
                     |
     +---------------+-----------------------------------------+
     |               Y                                         |
     |    -------> IP Stack                                    |
     |   |              |                                      |
     |   |              Y                                      |
     |   |              Y                                      |
     |   ^              |                                      |
     |   |  / ----------> Forwarding ->                        |
     |   ^ /                           |                       |
     |   |/                            Y                       |
     |   |                             |                       |
     |   ^                             Y          /-qdisc1-\   |
     |   |                            Egress     /--qdisc2--\  |
  --->->Ingress                       Classifier ---qdisc3---- | ->
     |   Qdisc                                   \__qdisc4__/  |
     |                                            \-qdiscN_/   |
     |                                                         |
     +----------------------------------------------------------+

```



tc 使用帮助
NAME

    tc # − 显示/操作 流量控制的设置

SYNOPSIS

    tc qdisc [ add | change | replace | link ] dev DEV [ parent qdisc-id | root ] [ handle qdisc-id ] qdisc [ qdisc specific parameters ]

    tc class [ add | change | replace ] dev DEV parent qdisc-id [ classid class-id ] qdisc [ qdisc specific parameters ]

    tc filter [ add | change | replace ] dev DEV [ parent qdisc-id | root ] protocol protocol prio priority filtertype [ filtertype specific parameters ] flowid flow-id

    tc [-s | -d ] qdisc show [ dev DEV ]

    tc [-s | -d ] class show dev DEV

    tc filter show dev DEV

DESCRIPTION

    tc 用于 Linux内核 的流量控制配置。流量控制包含以下内容:
    SHAPING
        当流量被重塑时，网络的传输速率将会在控制之下。重塑不仅仅能够减低可用的网络
        带宽————它还能用于平滑网络突发的情况，从而改善网络的状况。重塑发生在
        egress 处。
    SCHEDULING
        通过重新调整 packets 的传输调度策略，这可能能够改善网络的交互性，同时仍然
        保证批量传输的带宽。重新排序也被称为优先级排序，只发生在 egress 处。
    POLICING
        当流量控制被重塑时，policing 也会参与到流量控制中。 Policiing 仅仅发生在
        ingress。
    DROPPING
        可以设置流量的带宽，当流量超过设定值时，则立即丢弃。egress 和 ingress 处
        均发生。

QDISCS

    qdisc 是 queueing discipline 的简称， qdisc 是理解流量控制的基础。每当内核
    需要发送一个 packet 到接口时，packet 都会入队到与接口绑定的 qdisc。紧接着，
    内核会从 qdisc 获取尽可能多的数据包，并将它们提供给网络适配器驱动程序。

    一个简单的 qdisc 是 pfifo 型，它根本不进行任何处理，是一个纯粹的先进先出队
    列。但是，当网络接口无法立即处理流量时，它会存储流量。
    CLASSES
    一些 qdiscs 可以包含类 ———— 流量可以排队到内部任意某个 class 里，也就是在
    qdiscs 中。当内核试图从一个 具有类的qdiscs 中取出一个 packet
    时，这个 packet 可能来自于 qdiscs 中的任意一个类。

FILTERS

    一个 具有类的qdiscs 使用过滤器来决定 packet 究竟该入队到哪一个类。每当流量
    到达一个具有子类的类时，都需要对其进行分类。许多办法都可以达到这样一个效果，
    其中的一个方案就是使用过滤器。所有附加在类上的过滤器都会被调用，直到一个
    过滤器返回结果才停止，也有可能存在其他的标准根据具体的 qdiscs。

    又一点是值得注意的，就是过滤器是不知道发生了什么事情的，它们只是依附在
    qdiscs 上而已。

CLASSLESS QDISCS

    具有类的 qdisc 分别是：
    [p|b]fifo
        Simplest usable qdisc, pure First In, First Out behaviour. Limited in packets or in bytes.
    pfifo_fast
        Standard qdisc for ’Advanced Router’ enabled kernels. Consists of a three-band queue which honors Type of Service flags, as well as the priority that may be assigned to a packet.
        red Random Early Detection simulates physical congestion by randomly dropping packets when nearing configured bandwidth allocation. Well suited to very large bandwidth applications.
        sfq Stochastic Fairness Queueing reorders queued traffic so each ’session’ gets to send a packet in turn.
        tbf The Token Bucket Filter is suited for slowing traffic down to a precisely configured rate. Scales well to large bandwidths.

CONFIGURING CLASSLESS QDISCS

    在 无类的qdiscs 的情况下，无类的qdiscs 只能连接在设备的根目录下。完整语法：
    tc qdisc add dev DEV root QDISC QDISC-PARAMETERS
    在没有配置qdisc的情况下，pfifo_fast qdisc是自动默认值。

CLASSFUL QDISCS

    有类的qdisc 分别是：
        CBQ Class Based Queueing implements a rich linksharing hierarchy of classes. It contains shaping elements as well as prioritizing capabilities. Shaping is performed using link idle time calculations based on average packet size and underlying link bandwidth. The latter may be ill-defined for some interfaces.
        HTB The Hierarchy Token Bucket implements a rich linksharing hierarchy of classes with an emphasis on conforming to existing practices. HTB facilitates guaranteeing bandwidth to classes, while also allowing specification of upper limits to inter-class sharing. It contains shaping elements, based on TBF and can prioritize classes.
        PRIO The PRIO qdisc is a non-shaping container for a configurable number of classes which are dequeued in order. This allows for easy prioritization of traffic, where lower classes are only able to send if higher ones have no packets available. To facilitate configuration, Type Of Service bits are honored by default.

##### 相关模块

- nettem
- tbr
- sqf
- Class Based Queueding (CBQ)
- Hierachical Token Bucket (HTB)
- Multi Band Priority Queueding (PRIO)
- Hardware Multiqueue-aware Multi Band Queuing (MULTIQ)
- Random Early Detection(RED)
- Stochastic Fair Blue(SEQ)
- Token Bucket Filter(TBF)
- Network emulator(NETEM)
- Quick Fair Queueding sheduler(QFQ)
- Heavy-Hitter Filter(HHF)
- Ingress Qdisc
- Traffic-Control Index(TCINDEX)
- Universal 32bit comparisons w/ hashing (U32)
- Extended Matches
- U32 key

##### 带宽

```bash
bit or a bare number     Bits per second
kbit   Kilobits per second
mbit   Megabits per second
gbit   Gigabits per second
tbit   Terabits per second
bps    Bytes per second
kbps   Kilobytes per second
mbps   Megabytes per second
gbps   Gigabytes per second
tbps   Terabytes per second
```

带宽转换关系

```bash
1Mbit = 1024 Kbit = 1024 * 1024 bps => byte/s
```



##### netem

Netem 通过模拟广域网的特性，位测试协议提供了网络仿真功能。当前版本模拟传输延迟、丢包、重复包和包乱序。最好在网桥或路由器上使用netem。

###### 延迟

```bash
tc qdisc add dev eth0 root netem delay 100ms #固定延迟值
tc qdisc change dev eth0 root netem delay 100ms 10ms #100ms±10ms 随机变化
tc qdisc change dev eth0 root netem delay 100ms 10ms 25% #100ms±10ms 随机延迟出现的概率
tc qdisc change dev eth0 root netem delay 100ms 20ms distribution normal #延迟分布使用表格的方式来指定非均匀分布 （normal、pareto、paretonormal）
```

###### 丢包

tc 命令中以百分比的形式规定随机丢包的形式。最小的可能非零值为：1 / 0XFFFFFFFF = 0.0000000232%

```bash
tc qdisc change dev eth0 root netem loss 0.1%
tc qdisc change dev eth0 root netem loss 0.4% 25%
```

###### 重包

```bash
tc qdisc change dev eth0 root netem duplicate 1%
```

###### 数据包损坏

```bash
tc qdisc change dev eth0 root netem corrupt 0.1%
```

###### 包重排序

1. 指定排序 gap 固定序列，并每N个数据包重新排序一次。

   ```bash
   tc qdisc change dev eth0 root netem gap 5 delay 10ms
   ```



2. 重新排序更像现实生活。它会导致一定比例的数据包顺序错误。

   ```bash
   tc qdisc change dev eth0 root netem delay 10ms reorder 25% 50%
   tc qdisc change dev eth0 root netem delay 100ms 75ms
   ```

###### 速率控制

```bas
# DELAY_MS=40
# RATE_MBIT=10
# BUF_PKTS=33
# BDP_BYTES=$(echo "($DELAY_MS/1000.0)*($RATE_MBIT*1000000.0/8.0)" | bc -q -l)
# BDP_PKTS=$(echo "$BDP_BYTES/1500" | bc -q)
# LIMIT_PKTS=$(echo "$BDP_PKTS+$BUF_PKTS" | bc -q)
# tc qdisc replace dev eth0 root netem delay ${DELAY_MS}ms rate ${RATE_MBIT}Mbitlimit ${LIMIT_PKTS}
```

###### 其他

```bash
tc qdisc add dev eth0 root handle 1:0 netem delay 100ms
tc qdisc add dev eth0 parent 1:1 handle 10: tbf rate 256kbit buffer 1600 limit 3000
tc -s qdisc ls dev eth0
qdisc netem 1: limit 1000 delay 100.0ms
Sent 0 bytes 0 pkts (dropped 0, overlimits 0 )
qdisc tbf 10: rate 256Kbit burst 1599b lat 26.6ms
Sent 0 bytes 0 pkts (dropped 0, overlimits 0 )


tc qdisc add dev eth0 root handle 1:0 netem delay 100ms
tc qdisc add dev eth0 parent 1:1 handle 10: tbf rate 256kbit buffer 1600 limit 3000
tc -s qdisc ls dev eth0

qdisc netem 1: limit 1000 delay 100.0ms
Sent 0 bytes 0 pkts (dropped 0, overlimits 0 )
qdisc tbf 10: rate 256Kbit burst 1599b lat 26.6ms
Sent 0 bytes 0 pkts (dropped 0, overlimits 0 )
```



##### Ifb

```bash
                Userspace programs
                     ^
                     |
     +---------------+-----------------------------------------+
     |               Y                                         |
     |             IP Stack <-------------------------------------------------------|
     |                  |                                      |                    |
     |                  Y                                      |                    |
     |                  Y                                      |                    |
     |                  |                                      |                    |
     |                    Forwarding ->                        |                    |
     |                                 |                       |                    |
     |                                 Y                       |                    |
     |                                 |                       |                    |
     |                                 Y          /-qdisc1-\   |                    |
     |                                Egress     /--qdisc2--\  |                    |
  --->->Ingress                       Classifier ---qdisc3---- | ->                 |
     |   Qdisc                                   \__qdisc4__/  |                    |
     |     |                                      \-qdiscN_/   |                    |
     |     |                                                   |                    |
     +-----|----------------------------------------------------+                   |
           |                    ens33                                               |
           |                                                                        |
           |         +---------------+-----------------------------------------+    |
           |         |                                                         |    |
           |         |                                                         |    |
           |         |                                                         |    |
           |         |                                                         |    |
           |         |                                                         |    |
           |         |                                                         |    |
           |         |      / ----------> Forwarding ->                        |    |
           |         |     /                           |                       |    |
           |         |    /                            Y                       |    |
           |         |   |                             |                       |    |
           |         |   ^                             Y          /-qdisc1-\   |    |
           |         |   |                            Egress     /--qdisc2--\  |    |
           |--------->->Ingress                       Classifier ---qdisc3---- | ---|
                     |   Qdisc                                   \__qdisc4__/  |
                     |                                            \-qdiscN_/   |
                     |                                                         |
                     +----------------------------------------------------------+
                                            ifb

```

###### 实现原理

1. 将 ifb 虚拟网卡拉起来
2. 将 真实网卡(例如ens33) 的 Ingress 重定向到 ifb 的 Ingress
3. 对 ifb 的 Engress 进行控制

###### 配置示例

```bash
# 加载ifb驱动并创建ifb网卡(使用ifconfig -a 如果看到已有则无需该步骤)
modprobe ifb numifbs=1
# up网卡
ip link set dev ifb0 up

# 清除原有的根队列(根据实际情况操作,非必要)
tc qdisc del dev eth0 root 		2>/dev/null
tc qdisc del dev eth0 ingress 2>/dev/null
tc qdisc del dev ifb0 root 		2>/dev/null

#  将eth0的ingress流量全部重定向到 ifb0 处理
tc qdisc add dev eth0 handle ffff: ingress
tc filter add dev eth0 parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0

# eth0的出向限速:eth0添加根队列,使用htb,添加1:1类,使用htb
tc qdisc add dev eth0 root handle 1: htb r2q 625 default 65
tc class add dev eth0 parent 1: classid 1:1 htb rate 1000Mbit

# eth0的入向限速:ifb0添加根队列,使用htb,添加1:1类,使用htb
tc qdisc add dev ifb0 root handle 1: htb r2q 625 default 65
tc class add dev ifb0 parent 1: classid 1:1 htb rate 1000Mbit

# eth0的出向限速:eth0设备添加子类\对应的filter配置规则和子类的队列
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 10Mbit
tc filter add dev eth0 parent 1: protocol all prio 1 u32 match ip dst 192.168.0.2 classid 1:10
tc qdisc add dev eth0 parent 1:10 handle 10: sfq

# eth0的出向限速:eth0设备添加子类\对应的filter配置规则和子类的队列
tc class add dev eth0 parent 1:1 classid 1:11 htb rate 20Mbit
tc filter add dev eth0 parent 1: protocol all prio 1 u32 match ip dst 192.168.0.3 classid 1:11
tc qdisc add dev eth0 parent 1:11 handle 11: sfq


# eth0的入向限速:ifb0设备添加子类\对应的filter配置规则和子类的队列
tc class add dev ifb0 parent 1:1 classid 1:10 htb rate 10Mbit
tc filter add dev ifb0 parent 1: protocol all prio 1 u32 match ip src 192.168.0.2 classid 1:10
tc qdisc add dev ifb0 parent 1:10 handle 10: sfq

# eth0的入向限速:ifb0设备添加子类\对应的filter配置规则和子类的队列
tc class add dev ifb0 parent 1:1 classid 1:11 htb rate 20Mbit
tc filter add dev ifb0 parent 1: protocol all prio 1 u32 match ip src 192.168.0.3 classid 1:11
tc qdisc add dev ifb0 parent 1:11 handle 11: sfq

```

参考例子：https://www.freesion.com/article/51681050461/
