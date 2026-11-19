## 虚拟化

### KVM

### Docker

### Kubernetes
#### 概念

Kubernetes是容器集群管理系统，是一个开源的平台，可以实现容器集群的自动化部署、自动扩缩容、维护等功能。

##### Kubernetes特点

- **可移植**: 支持公有云，私有云，混合云，多重云（multi-cloud）
- **可扩展**: 模块化, 插件化, 可挂载, 可组合
- **自动化**: 自动部署，自动重启，自动复制，自动伸缩/扩展

##### 容器优势总结

- **快速创建/部署应用：**与VM虚拟机相比，容器镜像的创建更加容易。
- **持续开发、集成和部署：**提供可靠且频繁的容器镜像构建/部署，并使用快速和简单的回滚(由于镜像不可变性)。
- **开发和运行相分离：**在build或者release阶段创建容器镜像，使得应用和基础设施解耦。
- **开发，测试和生产环境一致性：**在本地或外网（生产环境）运行的一致性。
- **云平台或其他操作系统：**可以在 Ubuntu、RHEL、 CoreOS、on-prem、Google Container Engine或其它任何环境中运行。
- **Loosely coupled，分布式，弹性，微服务化：**应用程序分为更小的、独立的部件，可以动态部署和管理。
- **资源隔离**
- **资源利用：**更高效

##### Kubernetes作用

​		可以在物理或虚拟机的Kubernetes集群上运行容器化应用，Kubernetes能提供一个以“**容器为中心的基础架构**”，满足在生产环境中运行应用的一些常见需求，如：

- 多个进程（作为容器运行）协同工作。（Pod）
- 存储系统挂载
- Distributing secrets
- 应用健康检测
- 应用实例的复制
- Pod自动伸缩/扩展
- Naming and discovering
- 负载均衡
- 滚动更新
- 资源监控
- 日志访问
- 调试应用程序
- 提供认证和授权

#### Kubernetes组件

##### Master

Master组件提供集群的管理控制中心。

Master组件可以在集群中任何节点上运行。但是为了简单起见，通常在一台VM/机器上启动所有Master组件，并且不会在此VM/机器上运行用户容器。

##### kube-apiserver

kube-apiserver用于暴露Kubernetes API。任何的资源请求/调用操作都是通过kube-apiserver提供的接口进行。

##### ETCD

etcd是Kubernetes提供默认的存储系统，保存所有集群数据，使用时需要为etcd数据提供备份计划。

##### kube-controller-manager

kube-controller-manager运行管理控制器，它们是集群中处理常规任务的后台线程。逻辑上，每个控制器是一个单独的进程，但为了降低复杂性，它们都被编译成单个二进制文件，并在单个进程中运行。

这些控制器包括：

- 节点（Node）控制器。

- 副本（Replication）控制器：负责维护系统中每个副本中的pod。

- 端点（Endpoints）控制器：填充Endpoints对象（即连接Services＆Pods）。

- Service Account

  和Token控制器：为新的

  Namespace

   创建默认帐户访问API Token。

##### cloud-controller-manager

云控制器管理器负责与底层云提供商的平台交互。云控制器管理器是Kubernetes版本1.6中引入的，目前还是Alpha的功能。

云控制器管理器仅运行云提供商特定的（controller loops）控制器循环。可以通过将`--cloud-provider` flag设置为external启动kube-controller-manager ，来禁用控制器循环。

cloud-controller-manager 具体功能：

- 节点（Node）控制器

- 路由（Route）控制器

- Service控制器

- 卷（Volume）控制器



##### kube-scheduler

kube-scheduler 监视新创建没有分配到Node的Pod，为Pod选择一个Node。

##### 插件 addons

插件（addon）是实现集群pod和Services功能的 。Pod由Deployments，ReplicationController等进行管理。Namespace 插件对象是在kube-system Namespace中创建。

##### DNS

虽然不严格要求使用插件，但Kubernetes集群都应该具有集群 DNS。

群集 DNS是一个DNS服务器，能够为 Kubernetes services提供 DNS记录。

由Kubernetes启动的容器自动将这个DNS服务器包含在他们的DNS searches中。



##### 用户界面

kube-ui提供集群状态基础信息查看，容器资源监控提供一个UI浏览监控数据。

##### Cluster-level Logging

Cluster-level logging，负责保存容器日志，搜索/查看日志。

##### 节点（Node）组件

节点组件运行在Node，提供Kubernetes运行时环境，以及维护Pod。

##### kubelet

kubelet是主要的节点代理，它会监视已分配给节点的pod，具体功能：

- 安装Pod所需的volume。
- 下载Pod的Secrets。
- Pod中运行的 docker（或experimentally，rkt）容器。
- 定期执行容器健康检查。
- Reports the status of the pod back to the rest of the system, by creating a *mirror pod* if necessary.
- Reports the status of the node back to the rest of the system.

##### kube-proxy

kube-proxy通过在主机上维护网络规则并执行连接转发来实现Kubernetes服务抽象。

##### docker

docker用于运行容器。

##### RKT

rkt运行容器，作为docker工具的替代方案。

##### supervisord

supervisord是一个轻量级的监控系统，用于保障kubelet和docker运行。

##### fluentd

fluentd是一个守护进程，可提供cluster-level logging。



#### 安装

##### Kubernetes(1.18.5)

######  节点详情

```markdown
hostname	        ip地址              角色
k8s-node-master	192.168.1.25/24     master
k8s-node-1	     192.168.1.30/24      node
k8s-node-2	     192.168.1.35/24      node
```

######  设置静态IP

```bash
vim /etc/netplan/01-netcfg.yaml
```
 修改yaml(yet another markup language)内容
```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [192.168.1.25/24]
      nameservers:
           addresses: [114.114.114.114]

    enp0s8:
      dhcp4: yes

```
`netplan apply` 重启网络

###### 设置各个节点的hostname

```c
hostnamectl set-hostname "k8s-master"     // Run this command on master node
hostnamectl set-hostname "k8s-node-1"     // Run this command on node-0
hostnamectl set-hostname "k8s-node-2"     // Run this command on node-1
```

master节点添加host记录，其他节点类似
```bash
127.0.0.1       localhost
127.0.1.1       localhost
192.168.1.25    k8s-master
192.168.1.30    k8s-node-1
192.168.1.35    k8s-node-2

```
###### 备份更换源

```shell
mv sources.list sources.list.bak
```
```shell
vim /etc/apt/sources.list
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable
# deb-src [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable
```

###### 更新源安装常用软件

```shell
apt-get update && apt-get install -y tree net-tools  lrzsz  tree wget sysstat
```

###### 生成ssh密钥并分发给各节点

```shell
ssh-keygen -t rsa -P ""    #生成安全连接秘钥
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-node-1
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-node-2
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master
```
###### 安装`Ansible`（以下多台设备都用ansible）

```shell
apt-add-repository ppa:ansible/ansible
apt-get install ansible
```

设置/etc/ansible/hosts文件
```shell
[k8s]
k8s-master ansible_ssh_user=root
k8s-node-1 ansible_ssh_user=root
k8s-node-2 ansible_ssh_user=root

[node]
k8s-node-1 ansible_ssh_user=root
k8s-node-2 ansible_ssh_user=root
```

测试连接
```shell
ansible k8s -m ping
```
```shell
k8s-node-2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
k8s-node-1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
k8s-master | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```
如报错
```shell
FAILED! => {
"changed": false,
"module_stderr": "Shared connection to k8s-node-1  closed.\r\n",
"module_stdout": "/bin/sh: 1: /usr/bin/python: not found\r\n",
"msg": "MODULE FAILURE",
"rc": 127
}
```
由于未安装python2.7导致
```shell
apt-add-repository ppa:ansible/ansible #添加源
apt-get install python #安装python
python #查看版本
```
修改ansible.cfg 取消注释
```shell
vim /etc/ansible/ansible.cfg
command_warnings = False
```

###### node节点安装设置源常用软件

```shell
ansible node -m shell  -a 'mv /etc/apt/sources.list /etc/apt/sources.list_backup'
ansible node -m copy -a 'src=/etc/apt/sources.list dest=/etc/apt/'
ansible node -m apt -a 'update_cache=yes'
ansible node -m apt -a 'name=net-tools,lrzsz,tree,wget,sysstat,curl  state=present'
```

所有节点安装docker
卸载旧版本
```shell
ansible k8s -m apt -a 'name=docker,docker-engine,docker.io,containerd,runc state=absent'
```
更新源
```shell
ansible k8s -m apt -a 'update_cache=yes'
```
安装源添加工具
```shell
ansible k8s -m apt -a 'name=apt-transport-https,ca-certificates,curl,gnupg-agent,software-properties-common state=present'
```
添加密钥
```shell
ansible node -m shell  -a 'curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -'
```
检验密钥
```shell
ansible k8s -m shell  -a 'apt-key fingerprint 0EBFCD88'
ansible k8s -m shell -a 'add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"' #添加docker源
```
###### 更新源并安装docker-ce

```shell
ansible k8s -m apt -a 'update_cache=yes'
ansible k8s -m apt -a 'name=docker-ce state=present'
```
添加docker组
```shell
ansible k8s -m shell  -a 'groupadd docker'
```

为ubuntu设置docker使用权限(各节点分别操作)
```shell
su ubuntu
cd
sudo gpasswd -a ${USER} docker
```
启动docker并设置开机自启
```shell
ansible k8s -m systemd -a "name=docker  enabled=yes state=started"
```
查看docker信息和版本
```shell
ansible k8s -m shell  -a 'docker info '
ansible k8s -m shell  -a 'docker version '
k8s-master | SUCCESS | rc=0 >>
Client: Docker Engine - Community
 Version:           20.10.1
 API version:       1.41
 Go version:        go1.13.15
 Git commit:        831ebea
 Built:             Tue Dec 15 04:34:59 2020
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.1
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       f001486
  Built:            Tue Dec 15 04:32:40 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.3
  GitCommit:        269548fa27e0089a8b8278fc4fc781d7f65a939b
 runc:
  Version:          1.0.0-rc92
  GitCommit:        ff819c7e9184c13b7c2607fe6c30ae19403a7aff
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

```
给所有节点设置docker镜像源
```shell
ansible k8s -m shell  -a 'mkdir -p /etc/docker'
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://g9e8xuqp.mirror.aliyuncs.com"]
}
EOF
```
```shell
ansible node -m copy -a 'src=/etc/docker/daemon.json dest=/etc/docker/'
```

重启docker
```shell
ansible k8s -m systemd -a 'name=docker state=restarted'
```
查看docekr状态
```shell
ansible k8s -m shell  -a 'journalctl -u docker.server'
```

关闭swap 交换分区 （交换分区的读写和实际内存不一致）
```shell
ansible k8s -m shell  -a 'lsblk'
```
```shell
root@k8s-master:~# ansible k8s -m shell -a 'swapoff -a'
k8s-node-2 | SUCCESS | rc=0 >>
k8s-node-1 | SUCCESS | rc=0 >>
k8s-master | SUCCESS | rc=0 >>
```
修改vim /etc/fstab 注释掉swap这一行
```shell
root@k8s-master:~# cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/mapper/ubuntu--vg-root /               ext4    errors=remount-ro 0       1
#/dev/mapper/ubuntu--vg-swap_1 none            swap    sw              0       0

```
###### 开启ip转发

```shell
ansible k8s -m shell  -a "echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf "
ansible k8s -m shell  -a "sysctl -p"
```
```shell
root@k8s-master:~# ansible k8s -m shell  -a "sysctl -p"
k8s-node-2 | SUCCESS | rc=0 >>
net.ipv4.ip_forward = 1

k8s-node-1 | SUCCESS | rc=0 >>
net.ipv4.ip_forward = 1

k8s-master | SUCCESS | rc=0 >>
net.ipv4.ip_forward = 1
```

###### 安装k8s

```shell
ansible k8s -m apt -a 'name=apt-transport-https state=present'
ansible k8s -m shell -a ‘curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add –‘
tee /etc/apt/sources.list.d/kubernetes.list <<-'EOF'
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
ansible k8s -m apt -a 'update_cache=yes'
```
查看可安装版本
```shell
apt-cache madison kubelet
```
安装指定版本
```shell
ansible k8s -m apt -a 'name=kubelet=1.18.5-00,kubeadm=1.18.5-00,kubectl=1.18.5-00 state=present
```
启动
```shell
ansible k8s -m systemd -a "name=kubelet  enabled=yes state=started"
```
查看指定k8s版本需要哪些镜像
```shell
kubeadm config images list --kubernetes-version=v1.18.5
```
```shell
root@k8s-master:~# kubeadm config images list --kubernetes-version=v1.18.5
W1223 10:59:41.289737    4414 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
k8s.gcr.io/kube-apiserver:v1.18.5
k8s.gcr.io/kube-controller-manager:v1.18.5
k8s.gcr.io/kube-scheduler:v1.18.5
k8s.gcr.io/kube-proxy:v1.18.5
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.7
```
###### 通过dockerhub拉取镜像

```bash
#!/bin/bash
# Script For Quick Pull K8S Docker Images
# by Hellxz Zhang <hellxz001@foxmail.com>

KUBE_VERSION=v1.18.5
PAUSE_VERSION=3.2
CORE_DNS_VERSION=1.6.7
ETCD_VERSION=3.4.3-0

# pull kubernetes images from hub.docker.com
docker pull kubeimage/kube-proxy-amd64:$KUBE_VERSION
docker pull kubeimage/kube-controller-manager-amd64:$KUBE_VERSION
docker pull kubeimage/kube-apiserver-amd64:$KUBE_VERSION
docker pull kubeimage/kube-scheduler-amd64:$KUBE_VERSION
# pull aliyuncs mirror docker images
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$CORE_DNS_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION

# retag to k8s.gcr.io prefix
docker tag kubeimage/kube-proxy-amd64:$KUBE_VERSION  k8s.gcr.io/kube-proxy:$KUBE_VERSION
docker tag kubeimage/kube-controller-manager-amd64:$KUBE_VERSION k8s.gcr.io/kube-controller-manager:$KUBE_VERSION
docker tag kubeimage/kube-apiserver-amd64:$KUBE_VERSION k8s.gcr.io/kube-apiserver:$KUBE_VERSION
docker tag kubeimage/kube-scheduler-amd64:$KUBE_VERSION k8s.gcr.io/kube-scheduler:$KUBE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION k8s.gcr.io/pause:$PAUSE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$CORE_DNS_VERSION k8s.gcr.io/coredns:$CORE_DNS_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION k8s.gcr.io/etcd:$ETCD_VERSION

# untag origin tag, the images won't be delete.
docker rmi kubeimage/kube-proxy-amd64:$KUBE_VERSION
docker rmi kubeimage/kube-controller-manager-amd64:$KUBE_VERSION
docker rmi kubeimage/kube-apiserver-amd64:$KUBE_VERSION
docker rmi kubeimage/kube-scheduler-amd64:$KUBE_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$CORE_DNS_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION
```
```bash
vim k8s-pull.sh
```
修改查看所需的版本号并修改执行脚本
```bash
bash k8s-pull.sh
```
查看镜像
```bash
root@k8s-node-2:~# docker images
REPOSITORY                                                       TAG           IMAGE ID       CREATED         SIZE
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy   v1.18.14      8e6bca1d4e68   4 days ago      117MB
quay.io/coreos/flannel                                           v0.13.1-rc1   f03a23d55e57   4 weeks ago     64.6MB
k8s.gcr.io/kube-proxy                                            v1.18.5       a1daed4e2b60   5 months ago    117MB
k8s.gcr.io/kube-controller-manager                               v1.18.5       8d69eaf196dc   5 months ago    162MB
k8s.gcr.io/kube-apiserver                                        v1.18.5       08ca24f16874   5 months ago    173MB
k8s.gcr.io/kube-scheduler                                        v1.18.5       39d887c6621d   5 months ago    95.3MB
k8s.gcr.io/pause                                                 3.2           80d28bedfe5d   10 months ago   683kB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause        3.2           80d28bedfe5d   10 months ago   683kB
k8s.gcr.io/coredns                                               1.6.7         67da37a9a360   10 months ago   43.8MB
k8s.gcr.io/etcd                                                  3.4.3-0       303ce5db0e90   14 months ago   288MB

```
###### 所有节点拉取镜像并查看

```bash
ansible node  -m copy -a 'src=/root/k8s-pull.sh dest=/root/'
ansible node -m shell -a 'bash k8s-pull.sh'
ansible node -m shell -a 'docker images'
```

###### 初始化master节点

```bash
kubeadm init --apiserver-advertise-address 192.168.1.25 --pod-network-cidr=192.168.2.0/16 --ignore-preflight-errors=Swap --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```
```c
………
……….
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.25:6443 --token bscgxa.y3sz2g3zrrki6w86 \
--discovery-token-ca-cert-hash sha256:df97ed1ca7ab2bf15f1beee638807a05872b1864be23218151cda19b7c570e19

```
显示上面Your Kubernetes control-plane has initialized successfully!表示master节点初始化成功
为用户执行分配权限
```c
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
###### 安装网络组件

执行官网中网络初始化的命令，使用flannel网络模式

```bash
echo '199.232.28.133 raw.githubusercontent.com' >> /etc/hosts  #添加解析
```
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
若出错，则是官网网址被墙，直接下载到本地的yml文件执行kubectl apply -f kube-flannel.yml
OVER!!至此为止master节点配置完成。

###### 新增node节点

```bash
kubeadm join 192.168.1.25:6443 --token bscgxa.y3sz2g3zrrki6w86 \
--discovery-token-ca-cert-hash sha256:df97ed1ca7ab2bf15f1beee638807a05872b1864be23218151cda19b7c570e19
```

###### 查看已添加节点

```bash
kubectl get nodes   # 输出3个ready节点表示成功(可能需要等待一段时间)

```
###### CNI安装(master节点执行)

```bash
wget https://docs.projectcalico.org/v3.14/manifests/calico.yaml
```
修改IP为 172.20.0.0/16

```bash
kubectl apply -f calico.yaml
```
查看pod
```bash
kubectl get pod --all-namespaces
```
###### dashboard安装(UI)
1)创建dashboard服务

```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml
```
修改文件，新增30001 和NodePort
创建Pod
```bash
kubectl create -f recommended.yaml
```
查看，成功创建
```bash
kubectl get pods --all-namespaces
```

删除现有的dashboard服务，dashboard 服务的 namespace 是 kubernetes-dashboard，但是该服务的类型是ClusterIP，不便于我们通过浏览器访问，因此需要改成NodePort型的
```bash
kubectl get svc --all-namespaces（截图使用的已更换NodePort的图片）
```

删除
```bash
kubectl delete service kubernetes-dashboard --namespace=kubernetes-dashboard

```
创建dashboard配置文件
```bash
vim dashboard-svc.yaml
```

内容
```bash
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
```

执行刷新配置
```bash
kubectl apply -f dashboard-svc.yaml
```
再次查看服务
```bash
kubectl get svc --all-namespaces
```
访问dashboard服务，就要有访问权限，创建kubernetes-dashboard管理员角色

```bash
vim dashboard-svc-account.yaml
```
```bash
# 结果
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dashboard-admin
subjects:
  - kind: ServiceAccount
    name: dashboard-admin
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```
执行
```bash
kubectl apply -f dashboard-svc-account.yaml
```
获取Token
```bash
root@k8s-master:~/k8s/11# kubectl get secret -n kube-system |grep admin|awk '{print $1}'
dashboard-admin-token-qfj5t
root@k8s-master:~/k8s/11# kubectl describe secret dashboard-admin-token-qfj5t -n kube-system|grep '^token'|awk '{print $2}'
eyJhbGciOiJSUzI1NiIsImtpZCI6InM3cms1d2UwVVFyYkNZWEVKeVV5NWU5V1hDNVdUcjdFNXlic205THhIYXcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tcWZqNXQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiN2M3NDE3YmUtYTViZC00YmQ3LThhYmMtY2FjMjBjMWViODkxIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.ETCAeuIhFGughecyBtiyIFXrn-3OU7Z7UBgJkLHC6roKdhwGYmvmcDqzIwCYm_oR8NUug_lyGQN61zNIqkjbb0GPjUVni67vj0aTDVblVgzwiQGXd3J3DPVPxs3UGh1P5Ry7TC6P8rx6fsEw3rorIwG5qfP1PX8T3u9K4368YiDANwdoWdHG4n6A5iZdRlA-JwTrX-KxE1HMNnCDlNJ_Oa_j9lRrAmmjNmqghjFpfsGFXAwpDdbqlKWkrEleppxx_XaJu4fg8y7iI_J9WpeLlhe6wAtcZ21y2C6nNg6g2kRurXJqb_MIyJalPrbdd6OFNeZTP2XFLtV9G3puYVFivw
```

2)访问dashboard
浏览器访问https://192.168.1.25:30001  如下图所示
输入获取的token,登录

###### 错误解析
报错1：
```c
error execution phase preflight: couldn’t validate the identity of the API Server: abort connecting to API servers after timeout of 5m0s
```
```c

> kubeadm join ---
error execution phase preflight: couldn't validate the identity of the API Server: abort connecting to API servers after timeout of 5m0s
```
原因：master节点的token过期了
解决：创建新的token

```c
#得到token
>kubeadm token create
#得到discovery-token-ca-cert-hash
> openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

```
报错2
```c
calico/node is not ready: BIRD is not ready: BGP not established with 10.0.3.16,10.0.3.17
  Warning  BackOff  4m16s (x101 over 34m)  kubelet, ubuntu-2  Back-off restarting failed container
```
原因：calico部分网卡网络不通，enp0s8
解决：在BGP部分绑定enp0s3网卡

```c
# Specify interface
- name: IP_AUTODETECTION_METHOD
  value: "interface=enp0s3"
```
