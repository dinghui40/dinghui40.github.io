---
title: rainbond迁移数据库
date: 2022-06-10 17:23:31
tags:
categories: rainbond
---

气死人的一次
<!-- more -->

```
kubectl edit rainbondcluster -n rbd-system #进入修改数据库地址
kubectl delete pod -l name=rainbond-operator -n rbd-system  #重启控制节点
kubectl get pod -l name=rbd-api -n rbd-system -o yaml   #查看是否生效
最重要的一点数据备份
```

