---
title: docker二进制安装
date: 2022-06-22 13:39:26
tags:
categories: docker
---

本文主要提示docker创建systemctl启动脚本

<!-- more -->

# 部署docker

## 常用部署方式

删除docker环境残留

```
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

```
yum list installed | grep docker
yum -y remove
rm -rf /var/lib/docker
```

部署

```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum -y install docker-ce
```

配置镜像源

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://zhlkrut2.mirror.aliyuncs.com"]
}
EOF
```

启动

```
systemctl enable docker && systemctl start docker
```



## 二进制安装docker

## 介绍：

> 我个人是比较喜欢使用 docker 的，所以本次记录 docker 部署
>
> docker 二进制部署，基本适用于所有Liunx系统（不管是ubuntu或centos或---），都能一键启动

### 文件准备：

[docker软件包下载](https://download.docker.com/linux/static/stable/)

[阿里云镜像源加速](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors) 

### 开始部署：

解压 docker 二进制软件包  > 将文件移动到可执行目录下 > 启动

```bash
tar zxf docker-*.tgz
mv docker/* /usr/bin
sudo dockerd &
```

编写daemon.json文件

```
mkdir /etc/docker
vim /etc/docker/daemon.json
```

将 docker 加入 systemd 管理

```
vim /usr/lib/systemd/system/docker.service
```

```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP 
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

启动docker

```
systemctl daemon-reload 
systemctl enable docker
systemctl start docker
```



## yum源部署 docker

### centos｜ubuntu｜---

### 介绍：

> yum源部署过于简单，安装直接按照官网文档走就没有问题，没有需要特别注意的地方（摊手）
>
> 需要注意的可能是需要配置国内的yum源

### 准备：

[阿里yum源加速](https://developer.aliyun.com/mirror/)> 点击容器 > 选择 docker-ce

[Docker官网文档](https://docs.docker.com/engine/install/)
