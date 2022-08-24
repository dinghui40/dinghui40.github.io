---
title: ubuntu常用操作
date: 2022-06-22 10:19:36
tags:
categories: 系统优化
---

ubuntu的初始化，以及各种操作

<!-- more -->

| 步骤                     | 方法                                                         |
| :----------------------- | :----------------------------------------------------------- |
| 修改 root 密码           | 执行命令sudo passwd root                                     |
| 输入密码                 | 可以和 ubuntu 密码一致，也可以修改 (密码会让你输入两次)      |
| 修改 ssh 配置            | 执行命令sudo vi /etc/ssh/sshd_config                         |
| 修    改 PermitRootLogin | 进入 ssh 配置界面后找到PermitRootLogin，将它后面改为yes，保存 (按i进入编辑模式，编辑完esc退出，:w保存当前文件，:q退出) |
| 重启 ssh 服务            | 执行命令sudo service ssh restart                             |

