---
title: ceph存储
date: 2022-06-07 09:28:07
tags:
categories: 存储
---

## 部署Rook-ceph存储

<!-- more -->

### 前提条件

- 已部署Kubernetes集群，版本为 **v1.16**或更高版本。
- 集群最少拥有3个节点用来部署ceph存储。
- 建议的最低内核版本为**5.4.13**。如果内核版本低于 5.4.13则需要升级内核。
- 需要进行同步时间，官方要求0.05秒，并且做定时任务。
- 要求rainbond没有安装集群（停止在初始化化之前的一步，获取kubectl命令）

```
同步时间：  ntpdate -b ntp.aliyun.com;hwclock -w;

定时任务： 0 1 * * *  /usr/sbin/ntpdate -b ntp.aliyun.com;hwclock -w; 
  wget http://rainbond-pkg.oss-cn-shanghai.aliyuncs.com/offline/os-kernel/kernel_upgrade.tgz && tar xvf kernel_upgrade.tgz  && cd kernel_upgrade && sh kernel_upgrade.sh
```

- 安装lvm包

  - CentOS

    ```bash
    sudo yum install -y lvm2
    ```

  - Ubuntu

    ```bash
     sudo apt-get install -y lvm2
    ```

- 存储满足以下选项之一：

  - 原始设备（无分区或格式化文件系统）
  - 原始分区（无格式化的文件系统）

如果磁盘之前已经使用过，需要进行清理，使用以下脚本进行清理

```bash
#!/usr/bin/env bash
DISK="/dev/vdc"  #按需修改自己的盘符信息

# Zap the disk to a fresh, usable state (zap-all is important, b/c MBR has to be clean)

# You will have to run this step for all disks.
sgdisk --zap-all $DISK

# Clean hdds with dd
dd if=/dev/zero of="$DISK" bs=1M count=100 oflag=direct,dsync

# Clean disks such as ssd with blkdiscard instead of dd
blkdiscard $DISK

# These steps only have to be run once on each node
# If rook sets up osds using ceph-volume, teardown leaves some devices mapped that lock the disks.
ls /dev/mapper/ceph-* | xargs -I% -- dmsetup remove %

# ceph-volume setup can leave ceph-<UUID> directories in /dev and /dev/mapper (unnecessary clutter)
rm -rf /dev/ceph-*
rm -rf /dev/mapper/ceph--*

# Inform the OS of partition table changes
partprobe $DISK
```

### 部署Rook-Ceph

#### 部署 Rook Operator

```bash
git clone --single-branch --branch v1.9.0 https://github.com/rook/rook.git
cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

为Running则Operator部署成功

```bash
$ kubectl -n rook-ceph get pod
NAME                                 READY   STATUS    RESTARTS   AGE
rook-ceph-operator-b89545b4f-j64vk   1/1     Running   0          4m20s
```

#### 创建Ceph集群

- 给节点打标签

  为运行ceph-mon的节点打上：ceph-mon=enabled

  ```bash
  kubectl label nodes {kube-node1,kube-node2,kube-node3} ceph-mon=enabled
  ```

  为运行ceph-osd的节点，也就是存储节点，打上：ceph-osd=enabled

  ```bash
  kubectl label nodes {kube-node1,kube-node2,kube-node3} ceph-osd=enabled
  ```

  为运行ceph-mgr的节点，打上：ceph-mgr=enabled

  ceph-mgr最多只能运行2个

  ```bash
  kubectl label nodes {kube-node1,kube-node2} ceph-mgr=enabled
  ```

- 创建集群

  ```bash
  kubectl create -f cluster.yaml
  ```

#### 验证Ceph集群

通过命令行查看以下pod启动，表示成功：

```
kubectl get po -n rook-ceph
```

```bash
$ kubectl create -f toolbox.yaml
$ kubectl get po -l app=rook-ceph-tools -n rook-ceph
NAME                               READY   STATUS    RESTARTS   AGE
rook-ceph-tools-76876d788b-qtm4j   1/1     Running   0          77s

$ kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
$ ceph status
  cluster:
    id:     8bb6bbd4-ec90-4707-85d7-903551d08991
    health: HEALTH_OK #出现该字样则集群正常

  services:
    mon: 3 daemons, quorum a,b,c (age 77s)
    mgr: b(active, since 27s), standbys: a
    osd: 6 osds: 6 up (since 37s), 6 in (since 56s)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   31 MiB used, 750 GiB / 750 GiB avail #磁盘总容量，确认是否与实际容量相同
    pgs:     1 active+clean
```

### 提供存储

Rook-ceph提供了三种类型的存储：

- **[Block](https://rook.io/docs/rook/v1.8/ceph-block.html)**：创建由 Pod (RWO) 使用的块存储
- **[共享文件系统](https://rook.io/docs/rook/v1.8/ceph-filesystem.html)**：创建一个跨多个 Pod 共享的文件系统 (RWX)
- **[Object](https://rook.io/docs/rook/v1.8/ceph-object.html)**：创建一个可以在 Kubernetes 集群内部或外部访问的对象存储

#### 共享存储

- 打标签

为运行mds的节点打上：role=mds-node，通常为Ceph的三个节点

```bash
kubectl label nodes {kube-node1,kube-node2,kube-node3} role=mds-node
$ kubectl create -f filesystem.yaml
$ kubectl -n rook-ceph get pod -l app=rook-ceph-mds
NAME                                        READY   STATUS    RESTARTS   AGE
rook-ceph-mds-sharedfs-a-785c845496-2hcsz   1/1     Running   0          17s
rook-ceph-mds-sharedfs-b-87df7847-5rvx9     1/1     Running   0          16s
```

在 Rook 开始供应存储之前，需要根据文件系统创建一个 StorageClass。这是 Kubernetes 与 CSI 驱动程序互操作以创建持久卷所必需的。

```bash
$ kubectl create -f storageclass.yaml
$ kubectl get sc
NAME          PROVISIONER                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-cephfs   rook-ceph.cephfs.csi.ceph.com   Delete          Immediate           false                  72m
```

- 验证是否能够正常提供存储

  pod正常运行则存储正常挂载

```bash
kubectl create -f kube-registry.yaml
$ kubectl get po
NAME                             READY   STATUS    RESTARTS   AGE
kube-registry-66d4c7bf47-2h59z   1/1     Running   0          9m59s
kube-registry-66d4c7bf47-57dj4   1/1     Running   0          9m59s
kube-registry-66d4c7bf47-tgvn5   1/1     Running   0          9m59s
#删除验证pod
kubectl delete -f kube-registry.yaml
```

##### 假如包里没有文件则看如下：

```
创建共享存储
vi   filesystem.yaml

apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: replicated
      replicated:
        size: 3
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true

kubectl create -f filesystem.yaml
#########################################################################################
vi   storageclass.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  # clusterID is the namespace where the rook cluster is running
  # If you change this namespace, also change the namespace below where the secret namespaces are defined
  clusterID: rook-ceph

  # CephFS filesystem name into which the volume shall be created
  fsName: myfs

  # Ceph pool into which the volume shall be created
  # Required for provisionVolume: "true"
  pool: myfs-replicated

  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
  # in the same namespace as the cluster.
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

reclaimPolicy: Delete

kubectl create -f  storageclass.yaml
#########################################################################################
验证pod
vi  kube-registry.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
  namespace: kube-system
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-cephfs
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-registry
  namespace: kube-system
  labels:
    k8s-app: kube-registry
    kubernetes.io/cluster-service: "true"
spec:
  replicas: 3
  selector:
    matchLabels:
      k8s-app: kube-registry
  template:
    metadata:
      labels:
        k8s-app: kube-registry
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
      - name: registry
        image: registry:2
        imagePullPolicy: Always
        resources:
          limits:
            cpu: 100m
            memory: 100Mi
        env:
        # Configuration reference: https://docs.docker.com/registry/configuration/
        - name: REGISTRY_HTTP_ADDR
          value: :5000
        - name: REGISTRY_HTTP_SECRET
          value: "Ple4seCh4ngeThisN0tAVerySecretV4lue"
        - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
          value: /var/lib/registry
        volumeMounts:
        - name: image-store
          mountPath: /var/lib/registry
        ports:
        - containerPort: 5000
          name: registry
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: registry
        readinessProbe:
          httpGet:
            path: /
            port: registry
      volumes:
      - name: image-store
        persistentVolumeClaim:
          claimName: cephfs-pvc
          readOnly: false

kubectl create -f kube-registry.yaml

```

#### 块存储

```bash
kubectl create -f storageclass-block.yaml
```

```
vi  storageclass-block.yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
    # clusterID is the namespace where the rook cluster is running
    clusterID: rook-ceph
    # Ceph pool into which the RBD image shall be created
    pool: replicapool

    # (optional) mapOptions is a comma-separated list of map options.
    # For krbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd/#kernel-rbd-krbd-options
    # For nbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd-nbd/#options
    # mapOptions: lock_on_read,queue_depth=1024

    # (optional) unmapOptions is a comma-separated list of unmap options.
    # For krbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd/#kernel-rbd-krbd-options
    # For nbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd-nbd/#options
    # unmapOptions: force

    # RBD image format. Defaults to "2".
    imageFormat: "2"

    # RBD image features. Available for imageFormat: "2". CSI RBD currently supports only `layering` feature.
    imageFeatures: layering

    # The secrets contain Ceph admin credentials.
    csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
    csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
    csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
    csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

    # Specify the filesystem type of the volume. If not specified, csi-provisioner
    # will set default as `ext4`. Note that `xfs` is not recommended due to potential deadlock
    # in hyperconverged settings where the volume is mounted on the same node as the osds.
    csi.storage.k8s.io/fstype: ext4

# Delete the rbd volume when a PVC is deleted
reclaimPolicy: Delete

# Optional, if you want to add dynamic resize for PVC. Works for Kubernetes 1.14+
# For now only ext3, ext4, xfs resize support provided, like in Kubernetes itself.
allowVolumeExpansion: true
#########################################################/或（第一种没有验证所以列出了第二种）
vi    storageclass-block.yaml

apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: block-replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
    clusterID: rook-ceph
    pool: block-replicapool
    imageFormat: "2"
    imageFeatures: layering
    csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
    csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
    csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
    csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
    csi.storage.k8s.io/fstype: ext4

reclaimPolicy: Delete
allowVolumeExpansion: true
```

```
$ kubectl get storageclass  rook-ceph-block
NAME              PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-ceph-block   rook-ceph.rbd.csi.ceph.com   Delete          Immediate           true                   9s
获取到 storageclass 后已创建成功，在平台上有状态组件可选择块存储进行使用。
```

rook官网地址：https://rook.io/docs/rook/v1.9/quickstart.html  

初始化对接rainbond，修改自定义初始化（里面内容全部删除，替换以下内容）

```
spec:
  rainbondVolumeSpecRWX:
    storageClassName: rook-cephfs
```
