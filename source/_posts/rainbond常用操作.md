---
title: rainbond常用操作
date: 2022-06-07 09:32:19
tags:
---

忘了记得瞅瞅

<!-- more -->

# 操作命令

```
同步时间：  ntpdate -b ntp.aliyun.com;hwclock -w;
定时任务： 0 1 * * *  /usr/sbin/ntpdate -b ntp.aliyun.com;hwclock -w; 

更新内核版本：
wget http://rainbond-pkg.oss-cn-shanghai.aliyuncs.com/offline/os-kernel/kernel_upgrade.tgz && tar xvf kernel_upgrade.tgz  && cd kernel_upgrade && sh kernel_upgrade.sh
```



# 对接ceph集群更换镜像和添加变量

```
更换镜像和添加变量
kubectl edit rbdcomponents rbd-worker -n rbd-system
修改镜像和添加变量
registry.cn-hangzhou.aliyuncs.com/yangkaa/rbd-worker:v5.7.0-release
docker pull registry.cn-hangzhou.aliyuncs.com/yangkaa/rbd-worker:csi-dev
ENABLE_SUBPATH=true
env:
  - name: ENABLE_SUBPATH
    value: "true"
```

ceph存储路径查找：

然后进rbd-api容器，在/grdata目录下，对应的目录应该是/grdata/tenant/xxxx/service/xxxx/你创建的路径

然后进rbd-worker容器，在/grdata目录下，对应的目录应该是/grdata/tenant/xxxx/service/xxxx/你创建的路径

# 查看镜像仓库地址

```
kubectl describe   rainbondcluster  -n rbd-system
```

![image-20220608113210518](E:\github-ku\Blog\source\_posts\picture\image-20220608113210518.png)

503解决办法重启hub查看日志，并重新login仓库

清理镜像

https://t.goodrain.com/d/21-rbd-hub

# 对接存储更改成Retain

```
nfs:
  server:
  path: /rainbind
  mountOptions:
storageClass:
  reclaimPolicey: Retain
```

# 重新指定集群api地址


进行以下操作之前首先使用grctlconfig保存之前的yaml文件


- 修改集群api地址

```bash
 kubectl edit rainbondcluster -n rbd-system
 
   gatewayIngressIPs:
  - 10.193.121.112
```


- 保存之前的secret yaml文件

```bash
kubectl get secret rbd-api-client-cert -n rbd-system -o yaml > rbd-api-client-cert.yaml
kubectl get secret rbd-api-server-cert -n rbd-system -o yaml > rbd-api-server-cert.yaml
```


- 删除secret重新创建


```bash
kubectl delete secret rbd-api-server-cert rbd-api-client-cert -n rbd-system
kubectl delete po  -l name=rainbond-operator -n rbd-system  
kubectl delete po  -l name=rbd-api -n rbd-system
```

更改

```
kubectl edit rbdcomponents rbd-worker -n rbd-system
```

