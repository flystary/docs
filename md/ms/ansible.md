### Ansible

#### 概念

ansible基于Python开发，是基于 paramiko 开发的,并且基于模块化工作,基于ssh来和远
程主机通讯的

控制主机对被管节点的操作方式可分为两类，即`adhoc`和`playbook`：

- ad-hoc模式(点对点模式)
  　　使用单个模块，支持批量执行单条命令。ad-hoc 命令是一种可以快速输入的命令，而且不需要保存起来的命令。**就相当于bash中的一句话shell。**
- playbook模式(剧本模式)
  　　是Ansible主要管理方式，也是Ansible功能强大的关键所在。**playbook通过多个task集合完成一类功能**，

##### 常用模块

ping,command,shell,copy,file,fetch,cron,yum,apt,service,user,group,script,setup(filter,tree)

##### 命令执行过程

1. 加载自己的配置文件，默认`/etc/ansible/ansible.cfg`；
2. 查找对应的主机配置文件，找到要执行的主机或者组；
3. 加载自己对应的模块文件，如 command；
4. 通过ansible将模块或命令生成对应的临时py文件(python脚本)， 并将该文件传输至远程服务器；
5. 对应执行用户的家目录的`.ansible/tmp/XXX/XXX.PY`文件；
6. 给文件 +x 执行权限；
7. 执行并返回结果；
8. 删除临时py文件，`sleep 0`退出；

##### ansible 程序结构

安装目录如下(yum安装)：
　　配置文件目录：/etc/ansible/
　　执行文件目录：/usr/bin/
　　Lib库依赖目录：/usr/lib/pythonX.X/site-packages/ansible/
　　Help文档目录：/usr/share/doc/ansible-X.X.X/
　　Man文档目录：/usr/share/man/man1/

##### ansible配置文件

ansible 的配置文件为`/etc/ansible/ansible.cfg`，常见的参数：

```
library = /usr/share/ansible		#指向存放Ansible模块的目录，支持多个目录方式，只要用冒号（：）隔开就可以
forks = 5		#并发连接数，默认为5
sudo_user = root		#设置默认执行命令的用户
remote_port = 22		#指定连接被管节点的管理端口，默认为22端口，建议修改，能够更加安全
host_key_checking = False		#设置是否检查SSH主机的密钥，值为True/False。关闭后第一次连接不会提示配置实例
timeout = 60		#设置SSH连接的超时时间，单位为秒
log_path = /var/log/ansible.log		#指定一个存储ansible日志的文件（默认不记录日志
```

##### 命令集

`/usr/bin/ansible`　　Ansibe AD-Hoc 临时命令执行工具，常用于临时命令的执行
`/usr/bin/ansible-doc` 　Ansible 模块功能查看工具
`/usr/bin/ansible-galaxy`　　下载/上传优秀代码或Roles模块 的官网平台，基于网络的
`/usr/bin/ansible-playbook`　　Ansible 定制自动化的任务集编排工具
`/usr/bin/ansible-pull`　　Ansible远程执行命令的工具，拉取配置而非推送配置（使用较少，海量机器时使用，对运维的架构能力要求较高）
`/usr/bin/ansible-vault`　　Ansible 文件加密工具
`/usr/bin/ansible-console`　　Ansible基于Linux Consoble界面可与用户交互的命令执行工具

##### 命令格式

```shell
ansible <host-pattern> [-f forks] [-m module_name] [-a args]
```

`-a MODULE_ARGS`　　　#模块的参数，如果执行默认COMMAND的模块，即是命令参数，如： “date”，“pwd”等等
`-k`，`--ask-pass` #ask for SSH password。登录密码，提示输入SSH密码而不是假设基于密钥的验证
`--ask-su-pass` #ask for su password。su切换密码
`-K`，`--ask-sudo-pass` #ask for sudo password。提示密码使用sudo，sudo表示提权操作
`--ask-vault-pass` #ask for vault password。假设我们设定了加密的密码，则用该选项进行访问
`-B SECONDS` #后台运行超时时间
`-C` #模拟运行环境并进行预运行，可以进行查错测试
`-c CONNECTION` #连接类型使用
`-f FORKS` #并行任务数，默认为5
`-i INVENTORY` #指定主机清单的路径，默认为`/etc/ansible/hosts`
`--list-hosts` #查看有哪些主机组
`-m MODULE_NAME` #执行模块的名字，默认使用 command 模块，所以如果是只执行单一命令可以不用 -m参数
`-o` #压缩输出，尝试将所有结果在一行输出，一般针对收集工具使用
`-S` #用 su 命令
`-R SU_USER` #指定 su 的用户，默认为 root 用户
`-s` #用 sudo 命令
`-U SUDO_USER` #指定 sudo 到哪个用户，默认为 root 用户
`-T TIMEOUT` #指定 ssh 默认超时时间，默认为10s，也可在配置文件中修改
`-u REMOTE_USER` #远程用户，默认为 root 用户
`-v` #查看详细信息，同时支持`-vvv`，`-vvvv`可查看更详细信息

#### playbook

 ansible 用于配置，部署，和管理被控节点的剧本。

##### 格式

playbook由YMAL语言编写，YMAL格式类似JSON的文件格式

​       1、文件的第一行应该以 "---" (三个连字符)开始，表明YMAL文件的开始。
　　2、在同一行中，#之后的内容表示注释，类似于shell，python和ruby。
　　3、YMAL中的列表元素以”-”开头然后紧跟着一个空格，后面为元素内容。
　　4、同一个列表中的元素应该保持相同的缩进。否则会被当做错误处理。
　　5、play中hosts，variables，roles，tasks等对象的表示方法都是键值中间以":"分隔表示，":"后面还要增加一个空格。

##### 核心元素

`Hosts`：主机组；
`Tasks`：任务列表；
`Variables`：变量，设置方式有四种；
`Templates`：包含了模板语法的文本文件；
`Handlers`：由特定条件触发的任务；

##### 基本组件

`Hosts`：运行指定任务的目标主机
`remoute_user`：在远程主机上执行任务的用户；
`sudo_user`：
`tasks`：任务列表

```yaml
tasks：
　　　　　　– name: TASK_NAME
　　　　　　 module: arguments
　　　　　　 notify: HANDLER_NAME
　　　　　　 handlers:
　　　　　　– name: HANDLER_NAME
　　　　　　 module: arguments

```

`handlers`：任务，在特定条件下触发；接收到其它任务的通知时被触发；

　(1) 某任务的状态在运行后为changed时，可通过“notify”通知给相应的handlers；
　(2) 任务可以通过“tags“打标签，而后可在ansible-playbook命令上使用-t指定进行调用；

实例 nginx.yaml

```yaml
---
- hosts: web
  remote_user: root
  tasks:

    - name: install nginx
      yum: name=nginx state=present
    - name: copy nginx.conf
      copy: src=/tmp/nginx.conf dest=/etc/nginx/nginx.conf backup=yes
      notify: reload　　　　#当nginx.conf发生改变时，通知给相应的handlers
      tags: reloadnginx　　　#打标签
    - name: start nginx service
      service: name=nginx state=started
      tags: startnginx　　　#打标签

  handlers:　　#注意，前面没有-，是两个空格
    - name: reload
      service: name=nginx state=restarted　　#为了在进程中能看出来

```

测试标签

在里面已经打上了一个标签，所以可以直接引用标签。但是我们需要先把服务关闭，再来运行剧本并引用标签：

```bash
 ansible web -m shell -a 'systemctl stop nginx'
 ansible-playbook nginx.yml -t startnginx
```

测试notify

触发条件是配置文件被改变,例如改变nginx端口

```bash
[root@server ansible]# vim /tmp/nginx.conf
    listen       8080;
```

重新加载剧本

```bash
ansible-playbook nginx.yml -t reloadnginx
 #查看nginx端口已改变
[root@server ansible]# ansible web -m shell -a 'ss -ntlp | grep nginx'
192.168.37.122 | SUCCESS | rc=0 >>
LISTEN     0      128          *:8080                     *:*                   users:(("nginx",pid=2097,fd=6),("nginx",pid=2096,fd=6))

192.168.37.133 | SUCCESS | rc=0 >>
LISTEN     0      128          *:8080                     *:*                   users:(("nginx",pid=3061,fd=6),("nginx",pid=3060,fd=6))
```

##### variables

`variables`是变量，有四种定义方法

`facts` ：可直接调用,利用`setup`模块来获取，然后在playbook中直接使用

用户自定义变量

​	通过命令行传入：`ansible-playbook`命令的命令行中的`-e VARS, --extra-vars=VARS`，这样就可以直接把自定义的变量传入。

`直接定义`

```yaml
vars:
　　- var1: value1
　　- - var2: value2
```

###### 例子

使用全局替换(:% s/nginx/\{\{\ rpmname\ \}\}/g)把我们刚刚编辑的文件修改下

```yaml
---
- hosts: web
  remote_user: root
  tasks:

    - name: install {{ rpmname }}
      yum: name={{ rpmname }} state=present
    - name: copy {{ rpmname }}.conf
      copy: src=/tmp/{{ rpmname }}.conf dest=/etc/{{ rpmname }}/{{ rpmname }}.conf backup=yes
      notify: reload　　　　#当{{ rpmname }}.conf发生改变时，通知给相应的handlers
      tags: reload{{ rpmname }}　　　#打标签
    - name: start {{ rpmname }} service
      service: name={{ rpmname }} state=started
      tags: start{{ rpmname }}　　　#打标签

  handlers:　　#注意，前面没有-，是两个空格
    - name: reload
      service: name={{ rpmname }} state=restarted　　#为了在进程中能看出来
```

拷贝配置文件

在被监管的机器上安装什么服务的话，就直接在我们的server端上把该服务的配置文件拷贝到我们的`/tmp/`目录下

以安装`keepalived`服务为例

```bash
[root@server ansible]# cp /etc/keepalived/keepalived.conf /tmp/keepalived.conf
```

运行剧本，并以命令行传入变量名

```bash
[root@server ansible]# ansible-playbook nginx.yml -e rpmname=keepalived
```

修改剧本，直接在内部定义变量

```yaml
---
- hosts: web
  remote_user: root
  vars:
    - rpmname: keepalived
  tasks:
    - name: install {{ rpmname }}
      yum: name={{ rpmname }} state=present
    - name: copy {{ rpmname }}.conf
      copy: src=/tmp/{{ rpmname }}.conf dest=/etc/{{ rpmname }}/{{ rpmname }}.conf backup=yes
      notify: reload　　　　#当{{ rpmname }}.conf发生改变时，通知给相应的handlers
      tags: reload{{ rpmname }}　　　#打标签
    - name: start {{ rpmname }} service
      service: name={{ rpmname }} state=started
      tags: start{{ rpmname }}　　　#打标签

  handlers:　　#注意，前面没有-，是两个空格
    - name: reload
      service: name={{ rpmname }} state=restarted　　#为了在进程中能看出来
```

运行剧本

```bash
[root@server ansible]# ansible-playbook nginx.yml
```

 `通过roles传递变量`



`Host Inventory`

直接在主机清单中定义,

向不同的主机传递不同的变量：

```
　　IP/HOSTNAME varaiable=value var2=value2
```

向组中的主机传递相同的变量：

```
　　[groupname:vars]
　　variable=value
```

##### 模板 templates

模板是一个文本文件，嵌套有脚本（使用模板编程语言编写），Jinja2：Jinja2是python的一种模板语言，以Django的模板语言为原本

`模板支持`

```python
字符串：使用单引号或双引号；
　　数字：整数，浮点数；
　　列表：[item1, item2, ...]
　　元组：(item1, item2, ...)
　　字典：{key1:value1, key2:value2, ...}
　　布尔型：true/false
　　算术运算：
　　　　+, -, *, /, //, %, **
　　比较操作：
　　　　==, !=, >, >=, <, <=
　　逻辑运算：
　　　　and, or, not
```

通常来说，模板都是通过引用变量来运用的。

###### 举例

**①** **定义模板**
　　我们直接把之前定义的`/tmp/nginx.conf`改个名，然后编辑一下，就可以定义成我们的模板文件了：

```
[root@server ansible]# cd /tmp
[root@server tmp]# mv nginx.conf nginx.conf.j2
[root@server tmp]# vim nginx.conf.j2
    worker_processes  {{ ansible_processor_vcpus }};
    listen       {{ nginxport }};
```

**② 修改剧本**
　　我们现在需要去修改剧本来定义变量：

```
[root@server ansible]# vim nginx.yml
```

![](./img/44.png)

**③ 运行剧本**

```bash
 ansible-playbook nginx.yml -t reloadnginx
```

##### 条件测试

`when`语句：在task中使用，jinja2的语法格式。
举例如下：

```
tasks:
- name: install conf file to centos7
  template: src=files/nginx.conf.c7.j2
  when: ansible_distribution_major_version == "7"
- name: install conf file to centos6
  template: src=files/nginx.conf.c6.j2
  when: ansible_distribution_major_version == "6"
```

`循环`：迭代，需要重复执行的任务；
　　对迭代项的引用，固定变量名为"item"，而后，要在task中使用with_items给定要迭代的元素列表；
举例如下：

```
tasks:
- name: unstall web packages
  yum: name={{ item }} state=absent
  with_items:
  - httpd
  - php
  - php-mysql
```

##### 字典

ansible playbook 还支持字典功能。举例如下：

```
- name: install some packages
  yum: name={{ item }} state=present
  with_items:
    - nginx
    - memcached
    - php-fpm
- name: add some groups
  group: name={{ item }} state=present
  with_items:
    - group11
    - group12
    - group13
- name: add some users
  user: name={{ item.name }} group={{ item.group }} state=present
  with_items:
    - { name: 'user11', group: 'group11' }
    - { name: 'user12', group: 'group12' }
    - { name: 'user13', group: 'group13' }
```

##### roles

###### 简介

​	roles就是通过分别将变量(vars)、文件(file)、任务(tasks)、模块(modules)及处理器(handlers)放置于单独的目录中，并可以便捷地include它们的一种机制。角色一般用于基于主机构建服务的场景中，但也可以是用于构建守护进程等场景中。

###### 角色集合

`roles`
`mysql/`
`httpd`
`nginx/`
`files/`：存储由copy或script等模块调用的文件；
`tasks/`：此目录中至少应该有一个名为main.yml的文件，用于定义各task；其它的文件需要由main.yml进行“包含”调用；
`handlers/`：此目录中至少应该有一个名为main.yml的文件，用于定义各handler；其它的文件需要由main.yml进行“包含”调用；
`vars/`：此目录中至少应该有一个名为main.yml的文件，用于定义各variable；其它的文件需要由main.yml进行“包含”调用；
`templates/`：存储由template模块调用的模板文本；
`meta/`：此目录中至少应该有一个名为main.yml的文件，定义当前角色的特殊设定及其依赖关系；其它的文件需要由main.yml进行“包含”调用；
`default/`：此目录中至少应该有一个名为main.yml的文件，用于设定默认变量；

###### 实例

**1. 在roles目录下生成对应的目录结构**

```bash
[root@server ansible]# cd roles/
[root@server roles]# ls
[root@server roles]# mkdir -pv ./{nginx,mysql,httpd}/{files,templates,vars,tasks,handlers,meta,default}
[root@server roles]# tree
.
├── httpd
│   ├── default
│   ├── files
│   ├── handlers
│   ├── meta
│   ├── tasks
│   ├── templates
│   └── vars
├── mysql
│   ├── default
│   ├── files
│   ├── handlers
│   ├── meta
│   ├── tasks
│   ├── templates
│   └── vars
└── nginx
    ├── default
    ├── files
    ├── handlers
    ├── meta
    ├── tasks
    ├── templates
    └── vars

24 directories, 0 files
```

**2. 定义配置文件**
　　我们需要修改的配置文件为`/tasks/main.yml`，下面，我们就来修改一下：

```bash
[root@server roles]# vim nginx/tasks/main.yml
- name: cp
  copy: src=nginx-1.10.2-1.el7.ngx.x86_64.rpm dest=/tmp/nginx-1.10.2-1.el7.ngx.x86_64.rpm
- name: install
  yum: name=/tmp/nginx-1.10.2-1.el7.ngx.x86_64.rpm state=latest
- name: conf
  template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
  tags: nginxconf
  notify: new conf to reload
- name: start service
  service: name=nginx state=started enabled=true
```

**3. 放置我们所需要的文件到指定目录**
　　因为我们定义的角色已经有了新的组成方式，所以我们需要把文件都放到指定的位置，这样，才能让配置文件找到这些并进行加载。
　　rpm包放在`files`目录下，模板放在`templates`目录下：

```bash
[root@server nginx]# cp /tmp/nginx-1.10.2-1.el7.ngx.x86_64.rpm ./files/
[root@server nginx]# cp /tmp/nginx.conf.j2 ./templates/
[root@server nginx]# tree
.
├── default
├── files
│   └── nginx-1.10.2-1.el7.ngx.x86_64.rpm
├── handlers
├── meta
├── tasks
│   └── main.yml
├── templates
│   └── nginx.conf.j2
└── vars

7 directories, 3 files
```

**4. 修改变量文件**
　　我们在模板中定义的变量，也要去配置文件中加上：

```bash
[root@server nginx]# vim vars/main.yml
nginxprot: 9999
```

**5. 定义handlers文件**
　　我们在配置文件中定义了`notify`，所以我么也需要定义`handlers`，我们来修改配置文件：

```bash
[root@server nginx]# vim handlers/main.yml
- name: new conf to reload
  service: name=nginx state=restarted
```

**6. 定义剧本文件**
　　接下来，我们就来定义剧本文件，由于大部分设置我们都单独配置在了roles里面，所以，接下来剧本就只需要写一点点内容即可：

```bash
[root@server ansible]# vim roles.yml
- hosts: web
  remote_user: root
  roles:
    - nginx
```

**7. 启动服务**
　　剧本定义完成以后，我们就可以来启动服务了：

```bash
[root@server ansible]# ansible-playbook roles.yml
```

**8 .查看服务**

启动过后照例查看端口号：

```bash
[root@server ansible]# ansible web -m shell -a "ss -ntulp |grep 9999"
192.168.37.122 | SUCCESS | rc=0 >>
tcp    LISTEN     0      128       *:9999                  *:*                   users:(("nginx",pid=7831,fd=6),("nginx",pid=7830,fd=6),("nginx",pid=7829,fd=6))

192.168.37.133 | SUCCESS | rc=0 >>
tcp    LISTEN     0      128       *:9999                  *:*                   users:(("nginx",pid=9654,fd=6),("nginx",pid=9653,fd=6),("nginx",pid=9652,fd=6))
```