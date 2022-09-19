---
title: ceph所遇到的问题
date: 2022-06-07 14:20:21
tags:
categories: 存储
---

Ceph运维中所遇到的问题

<!-- more -->

```
ceph分配资源过小问题
查找
kubectl get sc rook-ceph
kubectl describe   sc rook-cephfs
加入
kubectl edit sc rook-cephfs -n rook-ceph
allowVolumeExpansion: True
命令行
kubectl exec -ti <pod名> -- df -kh |grep var
kubectl patch pvc <pvc名> -p '{"spec": {"resources": {"requests": {"storage": "5Gi"}}}}'
```

```
所有问题排查
https://access.redhat.com/documentation/zh-cn/red_hat_ceph_storage/5/pdf/troubleshooting_guide/red_hat_ceph_storage-5-troubleshooting_guide-zh-cn.pdf
```

视频：https://asciinema.org/a/351920

```
ceph遇到pod重启一直显示ContainerCreating
检查ceph集群是否正常
```

![d75c38524eeefc831768da71832cfef](E:\github-ku\Blog\source\_posts\picture\d75c38524eeefc831768da71832cfef.png)

![image-20220608113210518](E:\github-ku\Blog\source\_posts\picture\image-20220608113210518.png)

![cea37e7459570149f908dd725c9e416](E:\github-ku\Blog\source\_posts\picture\cea37e7459570149f908dd725c9e416.png)

![b037c2bf67599302d96a6cf745c86ee](E:\github-ku\Blog\source\_posts\picture\b037c2bf67599302d96a6cf745c86ee.png)

```
umount -lf /var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-1739b858-93b0-4ac4-b59e-773a84d3ea83/globalmount
```

#### 恢复排查

```
挂载：https://cloud.tencent.com/developer/article/1683695
挂载：https://blog.csdn.net/jycjyc/article/details/123025061?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0.pc_relevant_antiscanv2&spm=1001.2101.3001.4242.1&utm_relevant_index=3
https://www.qikqiak.com/k8strain/storage/ceph/   k8s训练营
安装：https://www.bookstack.cn/read/huweihuang-linux-notes/tools-ceph-fuse.md
安装：https://blog.51cto.com/u_11970509/2381262#:~:text=%E5%AE%89%E8%A3%85ceph%E5%AE%A2%E6%88%B7%E7%AB%AF%20%E5%9C%A8%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%B8%BB%E6%9C%BA%E4%B8%8A%E6%94%AF%E6%8C%81%E4%BB%A5%E4%B8%8B%E5%91%BD%E4%BB%A4%20wget%20-O%20%2Fetc%2Fyum.repos.d%2Fceph.repo%20https%3A%2F%2Fraw.githubusercontent.com%2Faishangwei%2Fceph-demo%2Fmaster%2Fceph-deploy%2Fceph.repo%20%E4%B8%8B%E8%BD%BDceph.repo%E9%95%9C%E5%83%8F%E6%BA%90%20yum,-s%20--name%20client.rbd%20%2F%2F%E6%9F%A5%E7%9C%8B%E9%9B%86%E7%BE%A4%E7%9A%84%E6%95%B4%E4%BD%93%E6%83%85%E5%86%B5%201.%202.%203.%204.
优化:
https://cloud.tencent.com/developer/article/1664654   日志位置
https://www.cnblogs.com/chuanzhang053/p/8710644.html   mysql目录查询
磁盘满了的问题：
https://blog.csdn.net/wanghuiict/article/details/61414122
https://www.51cto.com/article/476346.html
https://storage.it168.com/a2018/0703/3212/000003212605.shtml
https://bean-li.github.io/OSD-FULL/

打包所有镜像
docker save $(docker images | grep -v REPOSITORY | awk 'BEGIN{OFS=":";ORS=" "}{print $1,$2}') -o haha.tar

整理：
yum -y install epel-release
rpm -Uhv http://download.ceph.com/rpm-jewel/el7/noarch/ceph-release-1-1.el7.noarch.rpm
wget -O /etc/yum.repos.d/ceph.repo https://raw.githubusercontent.com/aishangwei/ceph-demo/master/ceph-deploy/ceph.repo
yum install -y ceph
mount | grep ceph  查找
ceph auth list 查找密钥
mount.ceph   IP和路径:/     挂载位置    -o name=admin,secret=密钥

du -sh *
fallocate -l 1G 文件名


排查：
https://www.bookstack.cn/read/ceph-en/288f7dc9097d5262.md
解决办法：https://www.cnblogs.com/st2021/p/14999988.html  （共享存储）
https://docs.ceph.com/en/latest/rados/operations/health-checks/#pool-near-full   查询

docker system prune -a
脚本：https://blog.csdn.net/wzj_110/article/details/107901012

http://www.zjjshen.xyz/2021/05/14/%E5%88%A0%E9%99%A4ceph%E6%8C%82%E8%BD%BD%E5%8D%B7%E4%B8%AD%E7%9A%84%E6%95%B0%E6%8D%AE%E5%90%8Eceph%E7%A9%BA%E9%97%B4%E6%B2%A1%E6%9C%89%E9%87%8A%E6%94%BE/

kubectl get pods --all-namespaces -o=json | jq -c '.items[] | {name: .metadata.name, namespace: .metadata.namespace, claimName:.spec.volumes[] | select( has ("persistentVolumeClaim") ).persistentVolumeClaim.claimName }'

```

```
损坏
https://blog.csdn.net/a13568hki/article/details/120264721
https://blog.51cto.com/wendashuai/2493183
https://blog.51cto.com/gravel/2517124
https://programmer.group/detailed-explanation-of-pg-state-of-distributed-storage-ceph.html
https://juejin.cn/post/6844903731784335374
https://www.tangyuecan.com/2020/02/17/%E5%9F%BA%E4%BA%8Ek8s%E6%90%AD%E5%BB%BAceph%E5%88%86%E9%83%A8%E7%BD%B2%E5%AD%98%E5%82%A8/
```

