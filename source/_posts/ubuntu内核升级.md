---
title: ubuntu内核升级
date: 2022-06-16 14:09:14
tags:
categories: 内核升级
---

## Ubuntu16.04

<!-- more -->

```
打开 http://kernel.ubuntu.com/~kernel-ppa/mainline/ 并选择列表中选择你需要的版本（以4.16.3为例）。
接下来，根据你的系统架构下载 如下.deb 文件：
Build for amd64 succeeded (see BUILD.LOG.amd64):
  linux-headers-4.16.3-041603_4.16.3-041603.201804190730_all.deb
  linux-headers-4.16.3-041603-generic_4.16.3-041603.201804190730_amd64.deb
  linux-image-4.16.3-041603-generic_4.16.3-041603.201804190730_amd64.deb
#安装后重启即可
$ sudo dpkg -i *.deb
```
