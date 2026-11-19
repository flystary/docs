## 监控

#### Zabbix

##### Linux环境说明

###### 安装要求

运行需要物理内存和磁盘空间。如果刚接触 Zabbix，128 MB 的物理内存和 256 MB 的可用磁盘空间可能是一个很好的起点。

```bash
[root@localhost ~]# cat /etc/redhat-release
CentOS Linux release 7.6.1810 (Core)
systemctl stop firewalld.service              #关闭防火墙
systemctl disable firewalld.service           #开机禁止启动防火墙
cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled                                  关闭selinux
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted

```

selinux setenforce 0 临时关闭

getenforce 结果为Disabled 为关闭 检查selinux是否关闭

##### 搭建LAMP环境

```bash
yum install -y httpd MariaDB-client  php php-mysql php-gd libjpeg* php-ldap php-odbc php-pear php-xml php-xmlrpc php-mhash

rpm -qa httpd php  MariaDB-client                         安装后检查应用版本
php-5.4.16-48.el7.x86_64
httpd-2.4.6-97.el7.centos.x86_64
MariaDB-client-10.5.11-1.el7.centos.x86_64
```

###### 编辑httpd

```bash
vim /etc/httpd/conf/httpd.conf

95 ServerName www.zabbix.com:80                             #修改主机名，URL
164     DirectoryIndex index.html index.php                  #修改首页文件格式
```

###### 编辑配置PHP，配置中国时区

```bash
vim /etc/php.ini

878 date.timezone = PRC
```

##### 创建数据库和用户

```shell
#新建zabbix用户
create user 'zabbix'@'%' identified by 'zabbix'
#新建zabbix数据库
create database zabbix character set utf8 collate utf8_bin;
#增加用户授权并刷新权限
grant all privileges on zabbix.* to 'zabbix'@'%' identified by 'zabbix';
flush privileges;
```

##### 安装zabbix,依赖包+组件

```shell
#依赖包安装 MariaDB-client为远程mysql工具
yum -y install   MariaDB-client net-snmp net-snmp-devel curl curl-devel libxml2 libxml2-devel libevent-devel.x86_64 javacc.noarch  javacc-javadoc.noarch javacc-maven-plugin.noarch javacc*

#安装php支持zabbix组件
yum install php-bcmath php-mbstring -y
#安装zabbix的yum源
rpm -ivh http://repo.zabbix.com/zabbix/4.0/rhel/7/x86_64/zabbix-release-4.0-1.el7.noarch.rpm

#安装zabbix组件
yum install zabbix-server-mysql zabbix-web-mysql -y

#导入sql数据到创建的数据库
zcat /usr/share/doc/zabbix-server-mysql-4.0.9/create.sql.gz | mysql -uzabbix -p -h 192.168.99.225 zabbix
```

Enter password: #导入数据到数据库zabbix中(最后一个zabbix是数据库zabbix)，且因为用户zabbix是%(任意主机)，所以登录时需要加上数据库主机ip(-h 192.168.99.200）密码是用户zabbix登陆密码zabbix

##### 修改zabbix配置文件

```bash
vim /etc/zabbix/zabbix_server.conf  #配置数据库密码

[root@localhost ~]# grep -n '^'[a-Z] /etc/zabbix/zabbix_server.conf
38:LogFile=/var/log/zabbix/zabbix_server.log
49:LogFileSize=0
72:PidFile=/var/run/zabbix/zabbix_server.pid
82:SocketDir=/var/run/zabbix
91:DBHost=192.168.99.225
100:DBName=zabbix
116:DBUser=zabbix
124:DBPassword=zabbix
139:DBPort=3306
356:SNMPTrapperFile=/var/log/snmptrap/snmptrap.log
471:Timeout=4
514:AlertScriptsPath=/usr/lib/zabbix/alertscripts
525:ExternalScripts=/usr/lib/zabbix/externalscripts
561:LogSlowQueries=3000
```

###### 修改时区

```bash
 vim /etc/httpd/conf.d/zabbix.conf
```

![image-20210702102348573](./img/image-20210702102348573.png)

##### 启动相关服务

```bash
#启动zabbix服务并加入开机自启动
systemctl enable zabbix-server
systemctl start zabbix-server

#启动httpd服务并加入开机自启动
systemctl start httpd
systemctl enable httpd
```

##### web配置

###### 浏览器访问  http://ip/zabbix

![image-20210702103010847](./img/53.png)



![image-20210702103147769](./img/769.png)



![image-20210702103218066](img/066.png)



![image-20210702103251077](img/1077.png)



![image-20210702103334243](img/34243.png)



![image-20210702103416277](img/16277.png)



U：`Admin` P：`zabbix`



![image-20210702103534657](img/534657.png)



![image-20210702103617959](img/7959.png)

##### zabbix-agent

###### 安装zabbix-agent并修改配置文件

```bash
yum install zabbix-agent -y
vim /etc/zabbix/zabbix_agentd.conf
grep -n '^'[a-Z] /etc/zabbix/zabbix_agentd.conf
13:PidFile=/var/run/zabbix/zabbix_agentd.pid
32:LogFile=/var/log/zabbix/zabbix_agentd.log
43:LogFileSize=0
98:Server=192.168.200.156
139:ServerActive=127.0.0.1
150:Hostname=Zabbix server
268:Include=/etc/zabbix/zabbix_agentd.d/*.conf
```

###### 启动zabbix-agent

```bash
systemctl start zabbix-agent.service      启动客户端
systemctl enable zabbix-agent.service    开机启动客户端
```

##### web中文化

zabbix网站页面进行汉化

![image-20210702142819869](img/9869.png)

移动到zabbix目录，并改为DejaVuSans.ttf

```
wget http://xze.197946.com/wenquanyiziti.zip
unzip wenquanyiziti.zip
cp ~/文泉驿点阵正黑.ttf  /usr/share/zabbix/assets/fonts/DejaVuSans.ttf
```

修改配置文件字体名称

```shell
 vim /usr/share/zabbix/include/defines.inc.php
```

![image-20210702142348025](img/025.png)

![image-20210702142431752](img/1752.png)



重启httpd，zabbix-server,zabbix-agent，

```shell
systemctl restart httpd
systemctl restart zabbix-server，zabbix-agent
```



![image-20210702142606647](img/06647.png)



#### Prometheus

##### 简介

##### 安装

##### 使用



#### OpenFalcon

##### 简介

###### 组件介绍

组件包括Agent

##### 安装
