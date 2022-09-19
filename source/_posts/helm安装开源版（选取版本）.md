---
title: helm安装开源版（选取版本）
date: 2022-06-07 09:29:23
tags:
categories: rainbond
---

rainbond-helm安装

<!-- more -->

```
kubectl create namespace rbd-system
helm repo add rainbond https://openchart.goodrain.com/goodrain/rainbond
helm repo update

helm install --set Cluster.gatewayIngressIPs=192.168.2.168 --set Cluster.enableHA=false --set Cluster.RWX.enable=true --set Cluster.RWX.config.storageClassName=rook-cephfs --set Cluster.RWO.enable=true --set Cluster.RWO.storageClassName=rook-cephfs --set Cluster.nodesForGateway[0].name=192.168.2.168 --set Cluster.nodesForGateway[0].externalIP=192.168.2.168 --set Cluster.nodesForGateway[0].internalIP=192.168.2.168 --set Cluster.installVersion=v5.7.0-release rainbond rainbond/rainbond-cluster -n rbd-system
```

```
删除helm集群
helm uninstall rainbond -n rbd-system 

kubectl get pvc -n rbd-system | grep -v NAME | awk '{print $1}' | xargs kubectl delete pvc -n rbd-system

kubectl get pv | grep rbd-system  | awk '{print $1}' | xargs kubectl delete pv

kubectl delete ns rbd-system

rm -rf /opt/rainbond
```

