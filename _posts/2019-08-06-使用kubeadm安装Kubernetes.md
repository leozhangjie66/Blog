---
layout: post
title: "使用kubeadm安装Kubernetes"
description: 使用kubeadm安装Kubernetes
modified: 2019-08-06
category: kubernetes
tags: [kubeadm, kubernetes, 安装]
imagefeature:
mathjax: false
chart:
comments: false
featured: false
---

# 使用kubeadm安装Kubernetes 1.15

> 本文仅做学习交流使用，如有侵权，立即删除。
> 参考文献：
> [https://blog.frognew.com/2019/07/kubeadm-install-kubernetes-1.15.html](https://blog.frognew.com/2019/07/kubeadm-install-kubernetes-1.15.html)  
> [https://zhuanlan.zhihu.com/p/35699988](https://zhuanlan.zhihu.com/p/35699988)  
> [https://github.com/opsnull/follow-me-install-kubernetes-cluster](https://github.com/opsnull/follow-me-install-kubernetes-cluster)  

## 1. 准备

### 1.1 环境信息

| 编号 |    主机别名     |     主机IP      | CPU、RAM |             系统              |         内核          |
| :--: | :------------: | :------------: | :------: | :---------------------------: | :-------------------: |
|  1   | kubeadm-node01 | 192.168.172.90 |   2U4G   | CentOS Linux release 7.6.1810 | 3.10.0-957.el7.x86_64 |
|  2   | kubeadm-node02 | 192.168.172.91 |   2U4G   | CentOS Linux release 7.6.1810 | 3.10.0-957.el7.x86_64 |

### 1.2 组件版本

// --TODO

### 1.3 系统初始化

- 修改主机名并设置hosts文件

```shell
# 更改主机名（分别在两个节点上执行对应的命令）
hostnamectl set-hostname kubeadm-node01
hostnamectl set-hostname kubeadm-node02

# 更改hosts文件（分别再两个节点的/etc/hosts文件中加入如下内容）
192.168.172.90 kubeadm-node01
192.168.172.91 kubeadm-node02
```

- 设置节点互信

```shell
# kubeadm-node01
ssh-keygen -t rsa
ssh-copy-id root@kubeadm-node02

# kubeadm-node02
ssh-keygen -t rsa
ssh-copy-id root@kubeadm-node01
```

- 配置yum源

centos系统自带的源有可能访问不了或者访问速度过慢，这里统一改为阿里源。

```bash
mkdir bak
mv *.repo bak
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
# 或者
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

# 配置好源后执行以下命令
yum clean all
yum makecache
```

- 安装依赖包

在每台机器上安装依赖包：

**`Centos:`**

```shell
yum install -y epel-release
yum install -y conntrack ntpdate ntp ipvsadm ipset jq iptables curl sysstat libseccomp wget
```

**`Ubuntu:`**

```shell
apt-get install -y conntrack ipvsadm ntp ipset jq iptables curl sysstat libseccomp
```

- 关闭swap分区

如果开启了 swap 分区，kubelet 会启动失败(可以通过将参数 `--fail-swap-on` 设置为 false 来忽略 swap on)，故需要在每台机器上关闭 swap 分区。同时注释 /etc/fstab 中相应的条目，防止开机自动挂载 swap 分区：

```bash
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab 
```

- 关闭防火墙

在每台机器上关闭防火墙，清理防火墙规则，设置默认转发策略：

```shell
systemctl stop firewalld
systemctl disable firewalld
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
iptables -P FORWARD ACCEPT
```

- 禁用SELINUX

关闭 SELinux，否则后续 K8S 挂载目录时可能报错 Permission denied：

```shell
# 临时、立即生效
setenforce 0

# 永久、重启生效
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

- 关闭 dnsmasq（可选）

linux 系统开启了 dnsmasq 后(如 GUI 环境)，将系统 DNS Server 设置为 127.0.0.1，这会导致 docker 容器无法解析域名，需要关闭它：

```shell
systemctl stop dnsmasq
systemctl disable dnsmasq
```

- 加载内核

```shell
modprobe ip_vs_rr
modprobe br_netfilter
```

- 优化内核参数

```shell
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF
sysctl -p /etc/sysctl.d/k8s.conf
```

> 必须关闭 tcp_tw_recycle，否则和 NAT 冲突，会导致服务不通；
> 关闭 IPV6，防止触发 docker BUG；


- 设置系统时区

```shell
# 调整系统 TimeZone
timedatectl set-timezone Asia/Shanghai

# 将当前的 UTC 时间写入硬件时钟
timedatectl set-local-rtc 0

# 重启依赖于系统时间的服务
systemctl restart rsyslog 
systemctl restart crond
```

- 关闭无关的服务

```shell
systemctl stop postfix
systemctl disable postfix
```

- 设置 rsyslogd 和 systemd journald

systemd 的 journald 是 Centos 7 缺省的日志记录工具，它记录了所有系统、内核、Service Unit 的日志。

相比 systemd，journald 记录的日志有如下优势：
1. 可以记录到内存或文件系统；(默认记录到内存，对应的位置为 /run/log/jounal)；
1. 可以限制占用的磁盘空间、保证磁盘剩余空间；
1. 可以限制日志文件大小、保存的时间；  
journald 默认将日志转发给 rsyslog，这会导致日志写了多份，/var/log/messages 中包含了太多无关日志，不方便后续查看，同时也影响系统性能。

```shell
mkdir /var/log/journal # 持久化保存日志的目录
mkdir /etc/systemd/journald.conf.d
cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
# 持久化保存到磁盘
Storage=persistent

# 压缩历史日志
Compress=yes

SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000

# 最大占用空间 10G
SystemMaxUse=10G

# 单日志文件最大 200M
SystemMaxFileSize=200M

# 日志保存时间 2 周
MaxRetentionSec=2week

# 不将日志转发到 syslog
ForwardToSyslog=no
EOF
systemctl restart systemd-journald
systemctl status systemd-journald
```

- 升级内核

CentOS 7.x 系统自带的 3.10.x 内核存在一些 Bugs，导致运行的 Docker、Kubernetes 不稳定，例如：
1. 高版本的 docker(1.13 以后) 启用了 3.10 kernel 实验支持的 kernel memory account 功能(无法关闭)，当节点压力大如频繁启动和停止容器时会导致 cgroup memory leak；
2. 网络设备引用计数泄漏，会导致类似于报错："kernel:unregister_netdevice: waiting for eth0 to become free. Usage count = 1";

解决方案如下：
1. 升级内核到 4.4.X 以上；
2. 或者，手动编译内核，disable CONFIG_MEMCG_KMEM 特性；
3. 或者，安装修复了该问题的 Docker 18.09.1 及以上的版本。但由于 kubelet 也会设置 kmem（它 vendor 了 runc），所以需要重新编译 kubelet 并指定 GOFLAGS="-tags=nokmem"；
  
``` shell
git clone --branch v1.14.1 --single-branch --depth 1 https://github.com/kubernetes/kubernetes
cd kubernetes
KUBE_GIT_VERSION=v1.14.1 ./build/run.sh make kubelet GOFLAGS="-tags=nokmem"
```

这里采用升级内核的解决办法：

``` shell
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
# 安装完成后检查 /boot/grub2/grub.cfg 中对应内核 menuentry 中是否包含 initrd16 配置，如果没有，再安装一次！
yum --enablerepo=elrepo-kernel install -y kernel-lt
# 设置开机从新内核启动
grub2-set-default 0
```

安装内核源文件（可选，在升级完内核并重启机器后执行）:

``` shell
# yum erase kernel-headers
yum --enablerepo=elrepo-kernel install kernel-lt-devel-$(uname -r) kernel-lt-headers-$(uname -r)
```

> 1. 系统内核相关参数参考：https://docs.openshift.com/enterprise/3.2/admin_guide/overcommit.html
> 2. 3.10.x 内核 kmem bugs 相关的讨论和解决办法：
>     1. https://github.com/kubernetes/kubernetes/issues/61937
>     2. https://support.mesosphere.com/s/article/Critical-Issue-KMEM-MSPH-2018-0006
>     3. https://pingcap.com/blog/try-to-fix-two-linux-kernel-bugs-while-testing-tidb-operator-in-k8s/

- 关闭NUMA

```shell
cp /etc/default/grub{,.bak}
vim /etc/default/grub # 在 GRUB_CMDLINE_LINUX 一行添加 `numa=off` 参数，如下所示：
diff /etc/default/grub.bak /etc/default/grub
6c6
< GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rhgb quiet"
---
> GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rhgb quiet numa=off"
```

重新生成 grub2 配置文件：

```shell
cp /boot/grub2/grub.cfg{,.bak}
grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 1.4 kube-proxy开启ipvs的前置条件

由于ipvs已经加入到了内核的主干，所以为kube-proxy开启ipvs的前提需要加载以下的内核模块：

```shell
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
```

在所有的Kubernetes节点node1和node2上执行以下脚本:

```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

上面脚本创建了的`/etc/sysconfig/modules/ipvs.modules`文件，保证在节点重启后能自动加载所需模块。 使用`lsmod | grep -e ip_vs -e nf_conntrack_ipv4`命令查看是否已经正确加载所需的内核模块。

接下来还需要确保各个节点上已经安装了ipset软件包`yum install ipset`。 为了便于查看ipvs的代理规则，最好安装一下管理工具ipvsadm `yum install ipvsadm`。

```shell
yum install -y ipset ipvsadm
```

如果以上前提条件如果不满足，则即使kube-proxy的配置开启了ipvs模式，也会退回到iptables模式。

### 1.5 安装Docker

Kubernetes从1.6开始使用CRI(Container Runtime Interface)容器运行时接口。默认的容器运行时仍然是Docker，使用的是kubelet中内置`dockershim CRI`实现。

添加docker的yum源：

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

查看docker版本

```shell
yum list docker-ce.x86_64  --showduplicates |sort -r
 * updates: mirrors.aliyun.com
Loading mirror speeds from cached hostfile
Loaded plugins: fastestmirror
 * extras: mirrors.aliyun.com
docker-ce.x86_64            3:18.09.7-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.6-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.5-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.4-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.3-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.2-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.1-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.0-3.el7                     docker-ce-stable
docker-ce.x86_64            18.06.3.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.06.2.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.06.1.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.06.0.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.03.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            18.03.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.12.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.12.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.09.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.09.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.06.2.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.06.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.06.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.03.3.ce-1.el7                    docker-ce-stable
docker-ce.x86_64            17.03.2.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.03.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.03.0.ce-1.el7.centos             docker-ce-stable
 * base: mirrors.aliyun.com
Available Packages

```

Kubernetes 1.15当前支持的docker版本列表是1.13.1, 17.03, 17.06, 17.09, 18.06, 18.09。 这里在各节点安装docker的18.09.7版本。

```shell
yum makecache fast
yum install -y --setopt=obsoletes=0 \
  docker-ce-18.09.7-3.el7

systemctl start docker
systemctl enable docker
systemctl status docker
```

确认一下`iptables filter`表中`FOWARD`链的默认策略(pllicy)为`ACCEPT`。

```shell
iptables -nvL
Chain INPUT (policy ACCEPT 10397 packets, 24M bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0

Chain OUTPUT (policy ACCEPT 7035 packets, 363K bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain DOCKER (1 references)
 pkts bytes target     prot opt in     out     source               destination

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER-ISOLATION-STAGE-2  all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DROP       all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-USER (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

### 1.6 修改docker cgroup driver为systemd

根据文档[CRI installation](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)中的内容，对于使用systemd作为init system的Linux的发行版，使用systemd作为docker的cgroup driver可以确保服务器节点在资源紧张的情况更加稳定，因此这里修改各个节点上docker的cgroup driver为systemd。

创建或修改`/etc/docker/daemon.json`：

```shell
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

重启docker：

```shell
systemctl restart docker

docker info | grep Cgroup
Cgroup Driver: systemd
```

## 2. 使用kubeadm部署Kubernetes

### 2.1 安装kubectl、kubelet和kubeadm

下面在各节点安装kubeadm和kubelet：

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF


# 开始安装
yum makecache fast
yum install -y kubelet kubeadm kubectl

Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * elrepo: hkg.mirror.rackspace.com
 * epel: sg.fedora.ipserverone.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package kubeadm.x86_64 0:1.15.0-0 will be installed
--> Processing Dependency: kubernetes-cni >= 0.7.5 for package: kubeadm-1.15.0-0.x86_64
--> Processing Dependency: cri-tools >= 1.11.0 for package: kubeadm-1.15.0-0.x86_64
---> Package kubectl.x86_64 0:1.15.0-0 will be installed
---> Package kubelet.x86_64 0:1.15.0-0 will be installed
--> Processing Dependency: socat for package: kubelet-1.15.0-0.x86_64
--> Running transaction check
---> Package cri-tools.x86_64 0:1.13.0-0 will be installed
---> Package kubernetes-cni.x86_64 0:0.7.5-0 will be installed
---> Package socat.x86_64 0:1.7.3.2-2.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

===================================================================================================================
 Package                       Arch                  Version                       Repository                 Size
===================================================================================================================
Installing:
 kubeadm                       x86_64                1.15.0-0                      kubernetes                8.9 M
 kubectl                       x86_64                1.15.0-0                      kubernetes                9.5 M
 kubelet                       x86_64                1.15.0-0                      kubernetes                 22 M
Installing for dependencies:
 cri-tools                     x86_64                1.13.0-0                      kubernetes                5.1 M
 kubernetes-cni                x86_64                0.7.5-0                       kubernetes                 10 M
 socat                         x86_64                1.7.3.2-2.el7                 base                      290 k

Transaction Summary
===================================================================================================================
Install  3 Packages (+3 Dependent packages)

Total download size: 55 M
Installed size: 251 M
Downloading packages:
warning: /var/cache/yum/x86_64/7/kubernetes/packages/7143f62ad72a1eb1849d5c1e9490567d405870d2c00ab2b577f1f3bdf9f547ba-kubeadm-1.15.0-0.x86_64.rpm: Header V4 RSA/SHA512 Signature, key ID 3e1ba8d5: NOKEY
Public key for 7143f62ad72a1eb1849d5c1e9490567d405870d2c00ab2b577f1f3bdf9f547ba-kubeadm-1.15.0-0.x86_64.rpm is not installed
(1/6): 7143f62ad72a1eb1849d5c1e9490567d405870d2c00ab2b577f1f3bdf9f547ba-kubeadm-1.15.0-0.x8 | 8.9 MB  00:00:02
(2/6): 14bfe6e75a9efc8eca3f638eb22c7e2ce759c67f95b43b16fae4ebabde1549f3-cri-tools-1.13.0-0. | 5.1 MB  00:00:03
(3/6): 3d5dd3e6a783afcd660f9954dec3999efa7e498cac2c14d63725fafa1b264f14-kubectl-1.15.0-0.x8 | 9.5 MB  00:00:01
(4/6): socat-1.7.3.2-2.el7.x86_64.rpm                                                       | 290 kB  00:00:01
(5/6): 548a0dcd865c16a50980420ddfa5fbccb8b59621179798e6dc905c9bf8af3b34-kubernetes-cni-0.7. |  10 MB  00:00:02
(6/6): 557c2f4e11a3ab262c72a52d240f2f440c63f539911ff5e05237904893fc36bb-kubelet-1.15.0-0.x8 |  22 MB  00:00:06
-------------------------------------------------------------------------------------------------------------------
Total                                                                              5.2 MB/s |  55 MB  00:00:10
Retrieving key from https://packages.cloud.google.com/yum/doc/yum-key.gpg
Importing GPG key 0xA7317B0F:
 Userid     : "Google Cloud Packages Automatic Signing Key <gc-team@google.com>"
 Fingerprint: d0bc 747f d8ca f711 7500 d6fa 3746 c208 a731 7b0f
 From       : https://packages.cloud.google.com/yum/doc/yum-key.gpg
Retrieving key from https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
Importing GPG key 0x3E1BA8D5:
 Userid     : "Google Cloud Packages RPM Signing Key <gc-team@google.com>"
 Fingerprint: 3749 e1ba 95a8 6ce0 5454 6ed2 f09c 394c 3e1b a8d5
 From       : https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : kubectl-1.15.0-0.x86_64                                                                         1/6
  Installing : socat-1.7.3.2-2.el7.x86_64                                                                      2/6
  Installing : kubernetes-cni-0.7.5-0.x86_64                                                                   3/6
  Installing : kubelet-1.15.0-0.x86_64                                                                         4/6
  Installing : cri-tools-1.13.0-0.x86_64                                                                       5/6
  Installing : kubeadm-1.15.0-0.x86_64                                                                         6/6
  Verifying  : kubeadm-1.15.0-0.x86_64                                                                         1/6
  Verifying  : kubelet-1.15.0-0.x86_64                                                                         2/6
  Verifying  : cri-tools-1.13.0-0.x86_64                                                                       3/6
  Verifying  : kubernetes-cni-0.7.5-0.x86_64                                                                   4/6
  Verifying  : socat-1.7.3.2-2.el7.x86_64                                                                      5/6
  Verifying  : kubectl-1.15.0-0.x86_64                                                                         6/6

Installed:
  kubeadm.x86_64 0:1.15.0-0             kubectl.x86_64 0:1.15.0-0             kubelet.x86_64 0:1.15.0-0

Dependency Installed:
  cri-tools.x86_64 0:1.13.0-0         kubernetes-cni.x86_64 0:0.7.5-0         socat.x86_64 0:1.7.3.2-2.el7

Complete!

```

从安装结果可以看出还安装了cri-tools, kubernetes-cni, socat三个依赖：

  - 官方从Kubernetes 1.14开始将cni依赖升级到了0.7.5版本
  - socat是kubelet的依赖
  - cri-tools是CRI(Container Runtime Interface)容器运行时接口的命令行工具 
运行kubelet --help可以看到原来kubelet的绝大多数命令行flag参数都被DEPRECATED了。

而官方推荐我们使用--config指定配置文件，并在配置文件中指定原来这些flag所配置的内容。具体内容可以查看这里[Set Kubelet parameters via a config file](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)。这也是Kubernetes为了支持动态Kubelet配置（Dynamic Kubelet Configuration）才这么做的，参考[Reconfigure a Node’s Kubelet in a Live Cluster](https://kubernetes.io/docs/tasks/administer-cluster/reconfigure-kubelet/)。

kubelet的配置文件必须是json或yaml格式，具体可查看[这里](https://github.com/kubernetes/kubernetes/blob/release-1.10/pkg/kubelet/apis/kubeletconfig/v1beta1/types.go)。

### 2.2 使用kubeadm init初始化集群

在各节点开机启动kubelet服务：

```bash
systemctl enable kubelet.service
```

使用`kubeadm config print init-defaults`可以打印集群初始化默认的使用的配置：

```bash
kubeadm config print init-defaults
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: kubeadm-node01
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.14.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

从默认的配置中可以看到，可以使用`imageRepository`定制在集群初始化时拉取k8s所需镜像的地址。基于默认配置定制出本次使用kubeadm初始化集群所需的配置文件`kubeadm.yaml`：

```bash
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.172.90
  bindPort: 6443
nodeRegistration:
  taints:
  - effect: PreferNoSchedule
    key: node-role.kubernetes.io/master
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.15.0
networking:
  podSubnet: 10.244.0.0/16
```

> 使用kubeadm默认配置初始化的集群，会在master节点打上`node-role.kubernetes.io/master:NoSchedule`的标签，阻止master节点接受调度运行工作负载。这里测试环境只有两个节点，所以将这个taint修改为`node-role.kubernetes.io/master:PreferNoSchedule`。

在开始初始化集群之前可以使用`kubeadm config images pull`预先在各个节点上拉取所k8s需要的docker镜像。

接下来使用kubeadm初始化集群，选择node1作为Master Node，在node1上执行下面的命令：

```bash
kubeadm init --config kubeadm.yaml --ignore-preflight-errors=Swap
[init] Using Kubernetes version: v1.15.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubeadm-node01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cl                                                               uster.local] and IPs [10.96.0.1 192.168.172.90]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kubeadm-node01 localhost] and IPs [192.168.172.90 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kubeadm-node01 localhost] and IPs [192.168.172.90 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up                                                                to 4m0s
[apiclient] All control plane components are healthy after 17.002718 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.15" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node kubeadm-node01 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kubeadm-node01 as control-plane by adding the taints [node-role.kubernetes.io/master:PreferNoSchedule]
[bootstrap-token] Using token: uxp5vg.s98jitvxw58vvtum
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
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

kubeadm join 192.168.172.90:6443 --token uxp5vg.s98jitvxw58vvtum \
    --discovery-token-ca-cert-hash sha256:4de0591206be04b4e83c3c5c26f68c3de66c77495813f56cc1a67816aacbe583
```

上面记录了完成的初始化输出的内容，根据输出的内容基本上可以看出手动初始化安装一个Kubernetes集群所需要的关键步骤。 其中有以下关键内容：




```bash
kubeadm join 192.168.172.90:6443 --token uxp5vg.s98jitvxw58vvtum --discovery-token-ca-cert-hash sha256:4de0591206be04b4e83c3c5c26f68c3de66c77495813f56cc1a67816aacbe583 --ignore-preflight-errors=Swap
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.15" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```


```bash
# kubectl get pod -n kube-system | grep kube-proxy
kube-proxy-kr6qd                         1/1     Running   0          30s
kube-proxy-whtw9                         1/1     Running   0          42s

# kubectl logs -n kube-system kube-proxy-kr6qd
I0714 19:49:38.786032       1 server_others.go:170] Using ipvs Proxier.
W0714 19:49:38.786342       1 proxier.go:401] IPVS scheduler not specified, use rr by default
I0714 19:49:38.786623       1 server.go:534] Version: v1.15.0
I0714 19:49:38.796560       1 conntrack.go:52] Setting nf_conntrack_max to 131072
I0714 19:49:38.797172       1 config.go:96] Starting endpoints config controller
I0714 19:49:38.797191       1 controller_utils.go:1029] Waiting for caches to sync for endpoints config controller
I0714 19:49:38.797524       1 config.go:187] Starting service config controller
I0714 19:49:38.797540       1 controller_utils.go:1029] Waiting for caches to sync for service config controller
I0714 19:49:38.897697       1 controller_utils.go:1036] Caches are synced for service config controller
I0714 19:49:38.897712       1 controller_utils.go:1036] Caches are synced for endpoints config controller
```


```bash
curl -O https://get.helm.sh/helm-v2.14.1-linux-amd64.tar.gz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 25.3M  100 25.3M    0     0  5699k      0  0:00:04  0:00:04 --:--:-- 5934k
[root@kubeadm-node01 ~]# ll
total 25920
-rw-------. 1 root root     1243 May 15 21:59 anaconda-ks.cfg
-rw-r--r--  1 root root 26532029 Jul 15 03:51 helm-v2.14.1-linux-amd64.tar.gz
drwxr-xr-x  2 root root       30 Jul 15 03:33 k8s
-rw-r--r--  1 root root      358 Jul 15 01:13 kubeadm.yaml
[root@kubeadm-node01 ~]# tar -zxvf helm-v2.14.1-linux-amd64.tar.gz
linux-amd64/
linux-amd64/LICENSE
linux-amd64/helm
linux-amd64/README.md
linux-amd64/tiller
[root@kubeadm-node01 ~]# cd linux-amd64/
[root@kubeadm-node01 linux-amd64]# cp helm /usr/local/bin/
[root@kubeadm-node01 linux-amd64]# ls
helm  LICENSE  README.md  tiller
[root@kubeadm-node01 linux-amd64]# ll
total 82716
-rwxr-xr-x 1 root root 41819776 Jun  6 05:19 helm
-rw-r--r-- 1 root root    11343 Jun  6 05:19 LICENSE
-rw-r--r-- 1 root root     3204 Jun  6 05:19 README.md
-rwxr-xr-x 1 root root 42864160 Jun  6 05:19 tiller
[root@kubeadm-node01 linux-amd64]# pwd
/root/linux-amd64
[root@kubeadm-node01 linux-amd64]# vi helm-rbac.yaml
[root@kubeadm-node01 linux-amd64]# kubectl create -f helm-rbac.yaml
serviceaccount/tiller created
clusterrolebinding.rbac.authorization.k8s.io/tiller created
[root@kubeadm-node01 linux-amd64]# helm init --service-account tiller --skip-refresh
Creating /root/.helm
Creating /root/.helm/repository
Creating /root/.helm/repository/cache
Creating /root/.helm/repository/local
Creating /root/.helm/plugins
Creating /root/.helm/starters
Creating /root/.helm/cache/archive
Creating /root/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /root/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation

```


```bash
# kubectl label node kubeadm-node01 node-role.kubernetes.io/edge=
node/kubeadm-node01 labeled
# kubectl get node
NAME             STATUS   ROLES         AGE   VERSION
kubeadm-node01   Ready    edge,master   31m   v1.15.0
kubeadm-node02   Ready    <none>        19m   v1.15.0
```

```bash
[root@kubeadm-node01 ~]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
[root@kubeadm-node01 ~]#
[root@kubeadm-node01 ~]# helm install stable/nginx-ingress \
> -n nginx-ingress \
> --namespace ingress-nginx  \
> -f ingress-nginx.yaml
NAME:   nginx-ingress
LAST DEPLOYED: Mon Jul 15 04:01:41 2019
NAMESPACE: ingress-nginx
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                                            READY  STATUS             RESTARTS  AGE
nginx-ingress-controller-9f88d564-rrmsf         0/1    ContainerCreating  0         1s
nginx-ingress-default-backend-7b8b45bd49-wq2g9  0/1    ContainerCreating  0         1s

==> v1/Service
NAME                           TYPE          CLUSTER-IP     EXTERNAL-IP  PORT(S)                     AGE
nginx-ingress-controller       LoadBalancer  10.110.41.102  <pending>    80:32053/TCP,443:31636/TCP  1s
nginx-ingress-default-backend  ClusterIP     10.100.20.145  <none>       80/TCP                      1s

==> v1/ServiceAccount
NAME           SECRETS  AGE
nginx-ingress  1        1s

==> v1beta1/ClusterRole
NAME           AGE
nginx-ingress  1s

==> v1beta1/ClusterRoleBinding
NAME           AGE
nginx-ingress  1s

==> v1beta1/Deployment
NAME                           READY  UP-TO-DATE  AVAILABLE  AGE
nginx-ingress-controller       0/1    1           0          1s
nginx-ingress-default-backend  0/1    1           0          1s

==> v1beta1/Role
NAME           AGE
nginx-ingress  1s

==> v1beta1/RoleBinding
NAME           AGE
nginx-ingress  1s


NOTES:
The nginx-ingress controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace ingress-nginx get services -o wide -w nginx-ingress-controller'

An example Ingress that makes use of the controller:

  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
    name: example
    namespace: foo
  spec:
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                serviceName: exampleService
                servicePort: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
        - hosts:
            - www.example.com
          secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```


```bash
helm install stable/kubernetes-dashboard -n kubernetes-dashboard --namespace kube-system  -f kubernetes-dashboard.yaml
NAME:   kubernetes-dashboard
LAST DEPLOYED: Mon Jul 15 04:05:45 2019
NAMESPACE: kube-system
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                                   READY  STATUS             RESTARTS  AGE
kubernetes-dashboard-848b8dd798-njs99  0/1    ContainerCreating  0         0s
kubernetes-dashboard-848b8dd798-wprb8  0/1    Terminating        0         48s

==> v1/Secret
NAME                  TYPE    DATA  AGE
kubernetes-dashboard  Opaque  0     0s

==> v1/Service
NAME                  TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)  AGE
kubernetes-dashboard  ClusterIP  10.99.208.237  <none>       443/TCP  0s

==> v1/ServiceAccount
NAME                  SECRETS  AGE
kubernetes-dashboard  1        0s

==> v1beta1/ClusterRoleBinding
NAME                  AGE
kubernetes-dashboard  0s

==> v1beta1/Deployment
NAME                  READY  UP-TO-DATE  AVAILABLE  AGE
kubernetes-dashboard  0/1    1           0          0s

==> v1beta1/Ingress
NAME                  HOSTS                  ADDRESS  PORTS  AGE
kubernetes-dashboard  192.168.172.90.nip.io  80, 443  0s


NOTES:
*********************************************************************************
*** PLEASE BE PATIENT: kubernetes-dashboard may take a few minutes to install ***
*********************************************************************************
From outside the cluster, the server URL(s) are:
     https://192.168.172.90.nip.io
```