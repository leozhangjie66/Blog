---
layout: post
title: "linux配置网卡eth0并修改为固定IP"
description: linux配置网卡eth0并修改为固定IP
modified: 2019-08-06
category: Linux
tags: [Linux, 网卡, IP, eth0]
imagefeature:
mathjax: false
chart:
comments: false
featured: false
---

>参考文件：  
>1.[https://blog.csdn.net/openbox2008/article/details/80051259](https://blog.csdn.net/openbox2008/article/details/80051259)  
>2.[https://segmentfault.com/a/1190000008743806](https://segmentfault.com/a/1190000008743806)  

## 编辑 grub 配置文件

执行`vi /etc/default/grub 或 vim /etc/sysconfig/grub`命令，在grub文件的`GRUB_CMDLINE_LINUX`中加入`net.ifnames=0 biosdevname=0`,最终如下：

```bash
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet net.ifnames=0 biosdevname=0"
GRUB_DISABLE_RECOVERY="true"
```

## 重新生成GRUB配置并更新内核

执行`grub2-mkconfig -o /boot/grub2/grub.cfg`，生成配置文件并更新内核，完成后重启系统，网卡名称已经变更为eth0

```bash
[root@localhost ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:a2:14:39 brd ff:ff:ff:ff:ff:ff
    inet 192.168.172.128/24 brd 192.168.172.255 scope global noprefixroute dynamic eth0
       valid_lft 1719sec preferred_lft 1719sec
    inet6 fe80::7e12:cd26:6f5c:ceaf/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

## 修改网卡配置文件

修改原网卡配置文件名称，并调整配置内容将IP配置为固定IP

```bash
cd /etc/sysconfig/network-scripts
mv ifcfg-ens33 ifcfg-eth0
vi ifcfg-eth0
```

```bash
TYPE="Ethernet"
DEVICE="eth0"
ONBOOT="yes"
BOOTPROTO="static"
IPADDR=192.168.172.50
NETMASK=255.255.255.0
GATEWAY=192.168.172.2
DNS1=8.8.8.8
DNS2=114.114.114.114
NAME="eth0"
UUID="392d2400-3972-487a-9064-629d095c3b5d"
IPV6INIT="no"
```

重启系统后即可生效