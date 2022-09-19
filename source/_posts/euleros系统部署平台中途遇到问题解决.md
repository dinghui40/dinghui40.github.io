---
title: euleros系统部署平台中途遇到问题解决
date: 2022-06-09 10:23:34
tags:
categories: rainbond问题排查
---

噢啦系统大成修炼阵法

<!-- more -->

首先因为euleros系统默认 /etc/ssh/sshd_config 文件里面访问权限大多数是系统默认关闭的，所以我们需要更改配置文件

```
#vi /etc/ssh/sshd_config   

Port 22                         #开放tcp隧道转发和22端口的访问，方便我们后期的远程连接访问
#AddressFamily any
ListenAddress 0.0.0.0
ListenAddress ::

LoginGraceTime 2m
PermitRootLogin yes
StrictModes yes

PasswordAuthentication yes

GSSAPIAuthentication yes

UsePAM yes

#AllowAgentForwarding yes
AllowTcpForwarding yes

TCPKeepAlive yes

PubkeyAuthentication yes
RSAAuthentication yes
IgnoreRhosts yes

StrictModes yes
AllowTcpForwarding yes
```

```
#init_node_offline.sh   文件内的（centos|redhat）这种地方需要添加euleros     
vi init_node_offline.sh
/centos搜索 （centos|redhat|euleros） 这样写 按esc 完后n 是跳转下一个目标
```

docker安装完成后安装k8s（rke方式）

安装k8s的时候首先检查docker是否能免密登录  ssh docker@IP 不能的话逐一排查

首先ssh-cpoy-id docker@IP 假如显示权限不足，则chown docker:docker /home/docker/.ssh 再次拷贝

安装完成之后进入网页访问

如果访问不到就是euleros系统的防火墙没开放端口

```
firewall-cmd --zone=public --add-port=7070/tcp --permanent
```

访问进去之后，注册登录

根据页面创建集群提示再把其他端口开放，完后再开始创建

华为yum源地址：

https://support.huaweicloud.com/csg_faq/csg_04_1209.html

内核更换

https://t.goodrain.com/t/topic/1305

EULER系统初始化

#允许其他用户连接

vi /etc/ssh/sshd_config 

```
LoginGraceTime 2m
PermitRootLogin yes
StrictModes yes
UsePAM no
```

#添加yum源

vi /etc/yum.repos.d/EulerOS.repo

```
[base]
name=EulerOS-2.0SP8 base
baseurl=http://mirrors.huaweicloud.com/euler/2.8/os/aarch64/
enabled=1
gpgcheck=1
gpgkey=http://mirrors.huaweicloud.com/euler/2.8/os/RPM-GPG-KEY-EulerOS
```

执行**yum clean all**清除原有yum缓存

执行**yum makecache**生成新的缓存

