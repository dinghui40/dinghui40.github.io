---
title: docker二进制安装
date: 2022-06-22 13:39:26
tags:
categories: docker
---

本文主要提示docker创建systemctl启动脚本

<!-- more -->

### .1.安装docker

```bash
1.解压二进制包
tar zxf docker-***.tgz

2.将可执行命令移动到系统路径
mv docker/* /usr/bin

3.创建配置文件
mkdir /etc/docker
vim /etc/docker/daemon.json
此处省略---
```

### 4.2.为docker创建systemctl启动脚本

```bash
1.编写启动脚本
vim /usr/lib/systemd/system/docker.service
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

2.启动docker
systemctl daemon-reload 
systemctl start docker
systemctl enable docker
```
