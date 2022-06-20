---
title: gfs排查
date: 2022-06-15 12:46:45
tags:
---

k8s-gfs

<!-- more -->

```
查看gfs状态：
kubectl get pod -n default  -o wide
kubectl exec -it pod -n default *3
gluster peer status
gluster volume status
gluster volume info

挂载gfs：
grctl查看存储
mount | grep 存储位置
mount -t glusterfs ip:/存储位置  /挂载位置
```

