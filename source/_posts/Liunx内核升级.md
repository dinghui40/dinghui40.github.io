---
title: liunx内核升级
date: 2022-06-16 14:00:45
tags:
categories: 内核升级
---

操作步骤

<!-- more -->

# centos-内核升级

## 小版本升级

**查看当前和可升级版本**

```bash
yum list kernel
```

**升级**

```bash
yum update kernel -y 
```

**重启并检查**

```bash
reboot 　

uname -r 
```

## 大版本升级

**载入公钥**

```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```

**升级安装ELRepo**

[ELRepo官网](http://elrepo.org/tiki/HomePage)

```bash
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
```

**载入elrepo-kernel元数据**

```bash
yum --disablerepo=\* --enablerepo=elrepo-kernel repolist
```

**查看可用的rpm包**

```bash
yum --disablerepo=\* --enablerepo=elrepo-kernel list kernel*
```

lt ：long term support，长期支持版本；

ml：mainline，主线版本；

**安装最新版本的kernel**

```bash
yum --disablerepo=\* --enablerepo=elrepo-kernel install  kernel-ml.x86_64  -y
```

**删除旧版本工具包**

```bash
yum remove kernel-tools-libs.x86_64 kernel-tools.x86_64  -y
```

**安装新版本工具包**

```bash
yum --disablerepo=\* --enablerepo=elrepo-kernel install kernel-ml-tools.x86_64  -y
```

**查看内核插入顺序**

```bash
awk -F \' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
```

默认新内核是从头插入，默认启动顺序也是从0开始（当前顺序还未生效）

其中文件 /etc/grub2.cfg 和 /boot/grub2/grub.cfg 内容一致。

**查看当前实际启动顺序**

```bash
grub2-editenv list
```

**设置默认启动**

```bash
grub2-set-default 'CentOS Linux (5.18.4-1.el7.elrepo.x86_64) 7 (Core)'

grub2-editenv list
```

**或者直接设置数值**

```bash
grub2-set-default 0　　#0代表当前第一行

grub2-editenv list
#saved_entry=0
```

**重启并检查**

```
reboot 

uname -r 
```



# ubuntu-内核升级

## 包指定版本升级

[内核包官网下载](http://kernel.ubuntu.com/~kernel-ppa/mainline/)

下载符合这两个格式的文件

```
linux-image-*-generic-*.deb
linux-modules-*-generic-*.deb
```

执行安装命令  > 重启 > 查看内核版本

```
sudo dpkg --install *.deb
sudo reboot

uname -r
```

