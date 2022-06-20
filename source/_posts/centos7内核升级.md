---
title: centos7内核升级
date: 2022-06-16 14:00:45
tags:
categories: 内核升级
---

操作步骤

<!-- more -->

**小版本升级**

**1. 查看当前和可升级版本**

```bash
yum list kernel
```

**2. 升级**

```bash
yum update kernel -y 
```

**3. 重启并检查**

```bash
reboot 　

uname -r 
```

**大版本升级**

**1. 载入公钥**

```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```

**2. 升级安装ELRepo**

```bash
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
```

**3. 载入elrepo-kernel元数据**

```bash
yum --disablerepo=\* --enablerepo=elrepo-kernel repolist
```

**4. 查看可用的rpm包**

```bash
yum --disablerepo=\* --enablerepo=elrepo-kernel list kernel*
```

lt ：long term support，长期支持版本；

ml：mainline，主线版本；

**5. 安装最新版本的kernel**

```bash
yum --disablerepo=\* --enablerepo=elrepo-kernel install  kernel-ml.x86_64  -y
```

**6. 删除旧版本工具包**

```bash
yum remove kernel-tools-libs.x86_64 kernel-tools.x86_64  -y
```

**7. 安装新版本工具包**

```bash
yum --disablerepo=\* --enablerepo=elrepo-kernel install kernel-ml-tools.x86_64  -y
```

**8. 查看内核插入顺序**

```bash
awk -F \' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
```

默认新内核是从头插入，默认启动顺序也是从0开始（当前顺序还未生效）

其中文件 /etc/grub2.cfg 和 /boot/grub2/grub.cfg 内容一致。

**9. 查看当前实际启动顺序**

```bash
grub2-editenv list
```

**10. 设置默认启动**

```bash
grub2-set-default 'CentOS Linux (5.18.4-1.el7.elrepo.x86_64) 7 (Core)'

grub2-editenv list
```

或者直接设置数值

```bash
grub2-set-default 0　　#0代表当前第一行

grub2-editenv list
#saved_entry=0
```

**11. 重启并检查**

```
reboot 

uname -r 
```

