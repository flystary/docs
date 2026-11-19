### tcpdump

#### 常用过滤规则

![516](./img/516.png)

1. option 可选参数：将在后边一一解释。
2. proto 类过滤器：根据协议进行过滤，可识别的关键词有： tcp, udp, icmp, ip, ip6, arp, rarp,ether,wlan, fddi, tr, decnet
3. type 类过滤器：可识别的关键词有：host, net, port, portrange，这些词后边需要再接参数。
4. direction 类过滤器：根据数据流向进行过滤，可识别的关键字有：src, dst，同时你可以使用逻辑运算符进行组合，比如 src or dst

#### 输出内容结构

```shell
21:26:49.013621 IP 172.20.20.1.15605 > 172.20.20.2.5920: Flags [P.], seq 49:97, ack 106048, win 4723, length 48

```

```bash
1. 第一列：时分秒毫秒 21:26:49.013621
2. 第二列：网络协议 IP
3. 第三列：发送方的ip地址+端口号，其中172.20.20.1是 ip，而15605 是端口号
4. 第四列：箭头 >， 表示数据流向
5. 第五列：接收方的ip地址+端口号，其中 172.20.20.2 是 ip，而5920 是端口号
6. 第六列：冒号
7. 第七列：数据包内容，包括Flags 标识符，seq 号，ack 号，win 窗口，数据长度 length，其中 [P.] 表示 PUSH 标志位为 1，更多标识符见下面
```

##### Flags标识符

- `[S]` : SYN（开始连接）
- `[P]` : PSH（推送数据）
- `[F]` : FIN （结束连接）
- `[R]` : RST（重置连接）
- `[.]` : 没有 Flag （意思是除上面四种类型外的其他情况，有可能是 ACK 也有可能是 URG）

#### 过滤规则

##### 基于IP地址过滤host

```bash
# 指定 host ip 进行过滤
tcpdump host 192.168.10.100
# 根据源ip进行过滤
tcpdump -i eth2 src 192.168.10.100
# 根据目标ip进行过滤
tcpdump -i eth2 dst 192.168.10.200
```

##### 基于网段过滤 net

```bash
# 指定网段
tcpdump net 192.168.10.0/24
# 根据源网段进行过滤
tcpdump src net 192.168
 # 根据目标网段进行过滤
tcpdump dst net 192.168

```

##### 基于端口过滤 port

```bash
# 指定特定端口
tcpdump port 8088
# 根据源端口进行过滤
tcpdump src port 8088
# 根据目标端口进行过滤
tcpdump dst port 8088
# 指定多个端口
tcpdump port 80 or port 8088
tcpdump port 80 or 8088
# 指定端口范围
tcpdump portrange 8000-8080
tcpdump src portrange 8000-8080
tcpdump dst portrange 8000-8080
# 默认端口 http == 80，https == 443
tcpdump tcp port http
```

##### 基于协议进行过滤 proto

常见的网络协议有：tcp, udp, icmp, http, ip,ipv6 等

```bash
tcpdump icmp
```

protocol 可选值：ip, ip6, arp, rarp, atalk, aarp, decnet, sca, lat, mopdl, moprc, iso, stp, ipx, or netbeui

##### 基本IP协议的版本进行过滤

```bash
tcpdump 'ip proto tcp'
# or
tcpdump ip proto 6
# or
tcpdump 'ip protochain tcp'
# or
tcpdump ip protochain 6
```

```bash
tcpdump 'ip && tcp'
tcpdump 'ip proto tcp'
# or
tcpdump 'ip6 && tcp'
tcpdump 'ip6 proto tcp'

```

#### 可选参数解析

##### 设置不解析域名

- `-n`：不把ip转化成域名，直接显示 ip，避免执行 DNS lookups 的过程，速度会快很多
- `-nn`：不把协议和端口号转化成名字，速度也会快很多。
- `-N`：不打印出host 的域名部分.。比如,，如果设置了此选现，tcpdump 将会打印'nic' 而不是 'nic.ddn.mil'.

##### 过滤结果输出到文件

 `-w` 参数后接一个以 `.pcap` 后缀命令的文件名，就可以将 tcpdump 抓到的数据保存到文件中。

```bash
tcpdump icmp -w icmp.pcap
```

##### 从文件中读取包数据

```bash
tcpdump icmp -r all.pcap
```

##### 控制详细内容的输出

- `-v`：产生详细的输出. 比如包的TTL，id标识，数据包长度，以及IP包的一些选项。同时它还会打开一些附加的包完整性检测，比如对IP或ICMP包头部的校验和。
- `-vv`：产生比-v更详细的输出. 比如NFS回应包中的附加域将会被打印, SMB数据包也会被完全解码。（摘自网络，目前我还未使用过）
- `-vvv`：产生比-vv更详细的输出。比如 telent 时所使用的SB, SE 选项将会被打印, 如果telnet同时使用的是图形界面，[手机游戏账号拍卖平台](https://www.fgba.net/)其相应的图形选项将会以16进制的方式打印出来（摘自网络，目前我还未使用过）

##### 控制时间的显示

- `-t`：在每行的输出中不输出时间
- `-tt`：在每行的输出中会输出时间戳
- `-ttt`：输出每两行打印的时间间隔(以毫秒为单位)
- `-tttt`：在每行打印的时间戳之前添加日期的打印（此种选项，输出的时间最直观）

##### 显示数据包的头部

- `-x`：以16进制的形式打印每个包的头部数据（但不包括数据链路层的头部）
- `-xx`：以16进制的形式打印每个包的头部数据（包括数据链路层的头部）
- `-X`：以16进制和 ASCII码形式打印出每个包的数据(但不包括连接层的头部)，这在分析一些新协议的数据包很方便。
- `-XX`：以16进制和 ASCII码形式打印出每个包的数据(包括连接层的头部)，这在分析一些新协议的数据包很方便。

##### 过滤指定网卡的数据包

- `-i`：指定要过滤的网卡接口，如果要查看所有网卡，可以 `-i any`

##### 过滤特定流向的数据包

- `-Q`： 选择是入方向还是出方向的数据包，可选项有：in, out, inout，也可以使用 --direction=[direction] 这种写法

##### 其他常用参数

- `-A`：以ASCII码方式显示每一个数据包(不显示链路层头部信息). 在抓取包含网页数据的数据包时, 可方便查看数据
- `-l` : 基于行的输出，便于你保存查看，或者交给其它工具分析
- `-q` : 简洁地打印输出。即打印很少的协议相关信息, 从而输出行都比较简短.
- `-c` : 捕获 count 个包 tcpdump 就退出
- `-s` : tcpdump 默认只会截取前 `96` 字节的内容，要想截取所有的报文内容，可以使用 `-s number`， `number` 就是你要截取的报文字节数，如果是 0 的话，表示截取报文全部内容。
- `-S` : 使用绝对序列号，而不是相对序列号
- `-C`：file-size，tcpdump 在把原始数据包直接保存到文件中之前, 检查此文件大小是否超过file-size. 如果超过了, 将关闭此文件,另创一个文件继续用于原始数据包的记录. 新创建的文件名与-w 选项指定的文件名一致, 但文件名后多了一个数字.该数字会从1开始随着新创建文件的增多而增加. file-size的单位是百万字节(nt: 这里指1,000,000个字节,并非1,048,576个字节, 后者是以1024字节为1k, 1024k字节为1M计算所得, 即1M=1024 ＊ 1024 ＝ 1,048,576)
- `-F`：使用file 文件作为过滤条件表达式的输入, 此时命令行上的输入将被忽略.

##### 对输出内容进行控制的参数

- `-D` : 显示所有可用网络接口的列表
- `-e` : 每行的打印输出中将包括数据包的数据链路层头部信息
- `-E` : 揭秘IPSEC数据
- `-L` ：列出指定网络接口所支持的数据链路层的类型后退出
- `-Z`：后接用户名，在抓包时会受到权限的限制。如果以root用户启动tcpdump，tcpdump将会有超级用户权限。
- `-d`：打印出易读的包匹配码
- `-dd`：以C语言的形式打印出包匹配码.
- `-ddd`：以十进制数的形式打印出包匹配码

#### 过滤规则组合

##### 逻辑运算符

- and：所有的条件都需要满足，也可以表示为 `&&`
- or：只要有一个条件满足就可以，也可以表示为 `||`
- not：取反，也可以使用 `!`

```bash
# 来自10.5.2.3，发往任意主机的3389端口的包
tcpdump src 10.5.2.3 and dst port 3389
```

```bash
tcpdump 'src 10.0.2.4 and (dst port 3389 or 22)'
```

##### 符号

- `=`：判断二者相等
- `==`：判断二者相等
- `!=`：判断二者不相等

关键字的接口

- if：表示网卡接口名、
- proc：表示进程名
- pid：表示进程 id
- svc：表示 service class
- dir：表示方向，in 和 out
- eproc：表示 effective process name
- epid：表示 effective process ID

```bash
tcpdump "( if=en0 and proc =nc ) || (if != en0 and dir=in)"
```

#### 特殊过滤规则

##### 使用tcpflags进行过滤

![tcpflag.png](./img/tcpflag.png.jpeg)

tcpdump 支持我们根据数据包的标志位进行过滤

```less
proto [ expr:size ]
```

- `proto`：可以是熟知的协议之一（如ip，arp，tcp，udp，icmp，ipv6）
- `expr`：可以是数值，也可以是一个表达式，表示与指定的协议头开始处的字节偏移量。
- `size`：是可选的，表示从字节偏移量开始取的字节数量。

1. tcpflags 别名常量，相当于13。代表着与指定的协议头开头相关的字节偏移量，也就是标志位，所以 tcp[tcpflags] 等价于 tcp[13] ，对应下图中的报文位置。

![img](./img/9fae.png)

2. tcp-fin, tcp-syn, tcp-rst, tcp-push, tcp-ack, tcp-urg 这些同样可以理解为别名常量，分别代表 1，2，4，8，16，32，64。

![img](./img/10d.png)

###### syn包

```bash
tcp[tcpflags] == tcp-syn
```

- 使用数字表示偏移量

```bash
tcpdump -i eth0 "tcp[13] & 2 != 0"
```

- 使用别名常量表示偏移量

```bash
$ tcpdump -i eth0 "tcp[tcpflags] & tcp-syn != 0"

```

- 使用混合写法

```bash
tcpdump -i eth0 "tcp[tcpflags] & 2 != 0"
# or
tcpdump -i eth0 "tcp[13] & tcp-syn != 0"
```



###### syn + ack 包

```bash
tcpdump -i eth0 'tcp[13] == 2 or tcp[13] == 16'
tcpdump -i eth0 'tcp[tcpflags] == tcp-syn or tcp[tcpflags] == tcp-ack'
tcpdump -i eth0 "tcp[tcpflags] & (tcp-syn|tcp-ack) != 0"

tcpdump -i eth0 'tcp[13] = 18'
# or
tcpdump -i eth0 'tcp[tcpflags] = 18'

```

icmp 别名常量

```bash
icmp-echoreply, icmp-unreach, icmp-sourcequench,
icmp-redirect, icmp-echo, icmp-routeradvert,
icmp-routersolicit, icmp-timx-ceed, icmp-paramprob,
icmp-tstamp, icmp-tstampreply,icmp-ireq,
icmp-ireqreply, icmp-maskreq, icmp-maskreply
```

##### 基于包大小进行过滤

```bash
tcpdump less 32
tcpdump greater 64
tcpdump <= 128
```

##### 基于Mac地址过滤

```bash
tcpdump ether host [ehost]
tcpdump ether dst [ehost]
tcpdump ether src [ehost]
```

##### 过滤通过指定网关的数据包

```bash
tcpdump gateway [host]
```

##### 过滤广播/多播数据包

```bash
tcpdump ether broadcast
tcpdump ether multicast

tcpdump ip broadcast
tcpdump ip multicast

tcpdump ip6 multicast
```



#### 高级使用

##### 抓取 HTTP 的 POST 请求解析

```bash
tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4]'
```

解析

- `tcp[n]`：表示 tcp 报文里 第 n 个字节
- `tcp[n:c]`：表示 tcp 报文里从第n个字节开始取 c 个字节，tcp[12:1] 表示从报文的第12个字节（因为有第0个字节，所以这里的12其实表示的是13）开始算起取一个字节，也就是 8 个bit。可以得知这 8 个bit 其实就是下图中的红框圈起来的位置，而在这里我们只要前面 4个bit，也就是实际数据在整个报文首部中的偏移量。

![img](./img/9762c01.png)

- `&`：是and 操作符，比如 `0011 & 0010 = 0010`
- `>>`：是位运算里的右移操作，比如 `0111 >> 2 = 0001`
- `0xf0`：是 10 进制的 240 的 16 进制表示，但对于位操作来说，10进制和16进制都将毫无意义，我们需要的是二进制，将其转换成二进制后是：11110000，这个数有什么特点呢？前面个 4bit 全部是 1，后面4个bit全部是0，往后看你就知道这个特点有什么用了。

```
tcp[12:1] & 0xf0
↓
10110110 && 11110000 = 10110000
```

```
tcp[12:1] & 0xf0) >> 2
↓
(tcp[12:1] & 0xf0) >> 4 ) << 2
↓
10110000  #前面 4 位（1011）才是正确的偏移量
↓
0x47455420
↓
tcp[((tcp[12:1] & 0xf0) >> 2):4]
```

```bas
0x47   -->   71    -->  G
0x45   -->   69    -->  E
0x54   -->   84    -->  T
0x20   -->   32    -->  空格

```

![img](./img/2d6459.png)

##### 从 HTTP 请求头中提取 HTTP 的 User-Agent

- 从 HTTP 请求头中提取 HTTP 的 User-Agent：

```bash
tcpdump -nn -A -s1500 -l | grep "User-Agent:"
```

- 通过 `egrep` 可以同时提取User-Agent 和主机名（或其他头文件）

```bash
tcpdump -nn -A -s1500 -l | egrep -i 'User-Agent:|Host:'
```

##### 抓取 HTTP GET 和 POST 请求

- 抓取 HTTP GET 请求包

```bash
tcpdump -s  -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
# or
tcpdump -vvAls0 | grep 'GET'
```

- 抓取 HTTP POST 请求包

```bash
tcpdump -s  -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354'
# or
tcpdump -vvAls0 | grep 'POST'
```

该方法不能保证抓取到 HTTP POST 有效数据流量，因为一个 POST 请求会被分割为多个 TCP 数据包

##### 找出发包数最多的 IP

```bash
tcpdump -nnn -t -c 200 | cut -f 1,2,3,4 -d '.' | sort | uniq -c | sort -nr | head -n 20
```

- **cut -f 1,2,3,4 -d '.'** : 以 `.` 为分隔符，打印出每行的前四列。即 IP 地址。
- **sort | uniq -c** : 排序并计数
- **sort -nr** : 按照数值大小逆向排序

##### 抓取 DNS 请求和响应

```bash
tcpdump -i any -s0 port 53 DNS 的默认端口是 53
```

##### 切割 pcap 文件

当抓取大量数据并写入文件时，可以自动切割为多个大小相同的文件。例如，下面的命令表示每 3600 秒创建一个新文件 `capture-(hour).pcap`，每个文件大小不超过 `200*1000000` 字节

```bash
tcpdump  -w /tmp/capture-%H.pcap -G 3600 -C 200
```

 `capture-{1-24}.pcap`，24 小时之后，之前的文件就会被覆盖。

##### 提取 HTTP POST 请求中的密码

从 HTTP POST 请求中提取密码和主机名

```bash
tcpdump -s 0 -A -n -l | egrep -i "POST /|pwd=|passwd=|password=|Host:"
```

##### 提取 HTTP 请求的 URL

提取 HTTP 请求的主机名和路径

```bash
tcpdump -s 0 -v -n -l | egrep -i "POST /|GET /|Host:"
```

##### 抓取 HTTP 有效数据包

抓取 80 端口的 HTTP 有效数据包，排除 TCP 连接建立过程的数据包（SYN / FIN / ACK）

```bash
tcpdump 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

```

maos操作技巧

```bash
$ ssh root@remotesystem 'tcpdump -s0 -c 1000 -nn -w - not port 22' | /Applications/Wireshark.app/Contents/MacOS/Wireshark -k -i -

$ ssh root@remotesystem 'tcpdump -s0 -c 1000 -nn -w - port 53' | /Applications/Wireshark.app/Contents/MacOS/Wireshark -k -i -

```
