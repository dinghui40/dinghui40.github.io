---
title: k8s二进制安装
date: 2022-06-16 17:20:50
tags:
categories: k8s
---

运行高效

<!-- more -->

## 部署介绍

> 本文介绍单机版部署k8s
>
> k8s集群部署请查看其他文章

## 包准备

[Etcd](https://github.com/etcd-io/etcd/releases)

[K8s-server](https://github.com/kubernetes/kubernetes/releases)

[cni文件](https://github.com/containernetworking/plugins/releases/)

# 环境准备

[证书准备](https://github.com/dinghui40/k8s/blob/master/kubernetes%E9%83%A8%E7%BD%B2/k8s%E8%AF%81%E4%B9%A6%E7%AD%BE%E5%8F%91.md)

[内核升级](https://github.com/dinghui40/k8s/blob/master/kubernetes%E9%83%A8%E7%BD%B2/Liunx%E5%86%85%E6%A0%B8%E5%8D%87%E7%BA%A7.md)

[部署docker](https://github.com/dinghui40/k8s/blob/master/kubernetes%E9%83%A8%E7%BD%B2/%E9%83%A8%E7%BD%B2%20docker.md)

> 一台机器
>
> | IP地址       | 主机名      | 网关        |
> | ------------ | ----------- | ----------- |
> | 192.168.1.10 | k8s-master1 | 192.168.1.1 |

修改主机名

```
hostnamectl set-hostname k8s-master1 && bash
hostnamectl status
```

修改hosts文件

```
cat >/etc/hosts<<'EOF'
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.10 k8s-master1
EOF
```

网络配置

> 这里需要注意的是 k8s 集群部署完成过后，主机将不能更换 IP 地址
>
> 更改为静态地址

查看网络配置`cat /etc/sysconfig/network-scripts/ifcfg-ens192`

```
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.1.10
GATEWAY=192.168.1.1
```

重启网络

```
service network restart
```

更换yum源

```
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum clean all
yum makecache
```

关闭防火墙

```
systemctl stop firewalld
systemctl disable --now firewalld
systemctl disable --now dnsmasq
systemctl disable --now NetworkManager
setenforce 0
```

关闭交换分区

```
swapoff -a
sysctl -w vm.swappiness=0
```

```
sed -i "s/\/dev\/mapper\/centos-swap/# \/dev\/mapper\/centos-swap/" /etc/fstab
cat /etc/fstab
```

关闭SeLinux

重启后可用`getenforce`确认SeLinux状态

```
sed -i "s/SELINUX=enforcing/SELINUX=disabled/" /etc/selinux/config
cat /etc/selinux/config
```

部署ntp时间同步

```
yum install ntp -y
```

启动

```
systemctl enable ntpd && systemctl start ntpd
```

时间同步（源随机挑选）

如果出现 `the NTP socket is in use, exiting` 报错则需要先执行 `service ntpd stop`

```
ntpdate cn.pool.ntp.org
```

查看机器时间

```
date
```

配置定时任务

使用此命令 `crontab  -e` 

```
* */1 * * * /usr/sbin/ntpdate	cn.pool.ntp.org
```

具自己需求准备基础依赖包

```
yum install net-tools checkpolicy gcc dkms foomatic openssh-server wget -y
```

加载 br_netfilter 模块

```
modprobe br_netfilter
lsmod |grep br_netfilter
```

修改内核参数

```
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1 
EOF

sysctl -p /etc/sysctl.d/k8s.conf
```

# 部署 etcd 集群

解压软件包

```
cd /root/k8s/
tar -xf etcd-v*-linux-amd64.tar.gz
cp -ar etcd-v*-linux-amd64/etcd* /usr/local/bin/
chmod +x /usr/local/bin/etcd*
mkdir -p /var/lib/etcd/default.etcd
```

创建配置文件

```
cat > /etc/kubernetes/etcd.conf <<EOF
#[Member]
ETCD_NAME="k8s-master1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.1.10:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.1.10:2379,http://127.0.0.1:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.1.10:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.1.10:2379"
ETCD_INITIAL_CLUSTER="k8s-master1=https://192.168.1.10:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

创建 ETCD 系统启动文件

```
cat > /usr/lib/systemd/system/etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
 
[Service]
Type=notify
EnvironmentFile=-/etc/kubernetes/etcd.conf
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \\
  --cert-file=/etc/kubernetes/pki/kubernetes.pem \\
  --key-file=/etc/kubernetes/pki/kubernetes-key.pem \\
  --trusted-ca-file=/etc/kubernetes/pki/ca.pem \\
  --peer-cert-file=/etc/kubernetes/pki/kubernetes.pem \\
  --peer-key-file=/etc/kubernetes/pki/kubernetes-key.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/pki/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
EOF
```

启动

```
systemctl daemon-reload && systemctl enable etcd.service && systemctl start etcd.service && systemctl status etcd && ETCDCTL_API=3
```

查看etcd endpoint状态

```
etcdctl --cacert=/etc/kubernetes/pki/ca.pem --cert=/etc/kubernetes/pki/kubernetes.pem --key=/etc/kubernetes/pki/kubernetes-key.pem --endpoints="https://192.168.1.10:2379" endpoint status --write-out=table
```

查看etcd endpoint 健康

```
etcdctl --write-out=table --cacert=/etc/kubernetes/pki/ca.pem --cert=/etc/kubernetes/pki/kubernetes.pem --key=/etc/kubernetes/pki/kubernetes-key.pem --endpoints=https://192.168.1.10:2379 endpoint health
```

```
etcdctl --cacert=/etc/kubernetes/pki/ca.pem --cert=/etc/kubernetes/pki/kubernetes.pem --key=/etc/kubernetes/pki/kubernetes-key.pem --endpoints="https://192.168.1.10:2379" endpoint health
```

查看 etcd 集群成员列表

```
etcdctl --cacert=/etc/kubernetes/pki/ca.pem --cert=/etc/kubernetes/pki/kubernetes.pem --key=/etc/kubernetes/pki/kubernetes-key.pem --endpoints="https://192.168.1.10:2379" member list
```

# 安装Kubernetes

解压软件包

```
tar zxvf kubernetes*-server-linux-amd64.tar.gz
cd kubernetes/server/bin/
cp kube-apiserver kube-controller-manager kube-scheduler kubectl kube-proxy kubelet /usr/local/bin/
mkdir /var/log/kubernetes
cd /root/k8s/
```

创建 token.csv 文件

```
cat > /etc/kubernetes/token.csv <<EOF
$(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

## 部署apiserver

创建apiserver的配置文件

```
cat > /etc/kubernetes/kube-apiserver.conf <<EOF
KUBE_APISERVER_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --anonymous-auth=false \\
  --bind-address=192.168.1.10 \\
  --secure-port=6443 \\
  --advertise-address=192.168.1.10 \\
  --insecure-port=0 \\
  --authorization-mode=Node,RBAC \\
  --runtime-config=api/all=true \\
  --enable-bootstrap-token-auth \\
  --service-cluster-ip-range=10.244.0.0/16 \\
  --token-auth-file=/etc/kubernetes/token.csv \\
  --service-node-port-range=30000-50000 \\
  --tls-cert-file=/etc/kubernetes/pki/kubernetes.pem  \\
  --tls-private-key-file=/etc/kubernetes/pki/kubernetes-key.pem \\
  --client-ca-file=/etc/kubernetes/pki/ca.pem \\
  --kubelet-client-certificate=/etc/kubernetes/pki/kubernetes.pem \\
  --kubelet-client-key=/etc/kubernetes/pki/kubernetes-key.pem \\
  --service-account-key-file=/etc/kubernetes/pki/ca-key.pem \\
  --service-account-signing-key-file=/etc/kubernetes/pki/ca-key.pem  \\
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \\
  --etcd-cafile=/etc/kubernetes/pki/ca.pem \\
  --etcd-certfile=/etc/kubernetes/pki/kubernetes.pem \\
  --etcd-keyfile=/etc/kubernetes/pki/kubernetes-key.pem \\
  --etcd-servers=https://192.168.1.10:2379 \\
  --enable-swagger-ui=true \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/kube-apiserver-audit.log \\
  --event-ttl=1h \\
  --alsologtostderr=true \\
  --logtostderr=false \\
  --log-dir=/var/log/kubernetes \\
  --v=4"
EOF
```

创建服务启动文件

```
vi /usr/lib/systemd/system/kube-apiserver.service
```

```
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=etcd.service
Wants=etcd.service
 
[Service]
EnvironmentFile=-/etc/kubernetes/kube-apiserver.conf
ExecStart=/usr/local/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
```

启动

```
systemctl daemon-reload && systemctl enable kube-apiserver && systemctl start kube-apiserver && systemctl status kube-apiserver
```

验证 (401为正常)

```
curl --insecure https://192.168.1.10:6443/
```

## 部署 kubectl 组件

置一个环境变量

```
export KUBECONFIG=/etc/kubernetes/admin.conf
```

配置安全上下文,设置集群参数

```
cd /etc/kubernetes/pki/
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.1.10:6443 --kubeconfig=kube.config
```

设置客户端认证参数

```
kubectl config set-credentials admin --client-certificate=admin.pem --client-key=admin-key.pem --embed-certs=true --kubeconfig=kube.config
```

设置上下文参数

```
kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=kube.config
```

设置当前上下文

```
kubectl config use-context kubernetes --kubeconfig=kube.config
mkdir ~/.kube -p
cp kube.config ~/.kube/config
cp kube.config /etc/kubernetes/admin.conf
```

授权 kubernetes 证书访问kubelet api 权限

```
kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes
```

查看集群组件状态

```
kubectl cluster-info
kubectl get componentstatuses
kubectl get all --all-namespaces
```

## 配置 kubectl 子命令补全

```
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion 
source <(kubectl completion bash)
kubectl completion bash > ~/.kube/completion.bash.inc
source '/root/.kube/completion.bash.inc' 
source $HOME/.bash_profile
```

## 部署 kube-controller-manager的

创建 kube-controller-manager 的 kubeconfig,设置集群参数

```
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.1.10:6443 --kubeconfig=kube-controller-manager.kubeconfig
```

设置客户端认证参数

```
kubectl config set-credentials system:kube-controller-manager --client-certificate=kube-controller-manager.pem --client-key=kube-controller-manager-key.pem --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig
```

设置上下文参数

```
kubectl config set-context system:kube-controller-manager --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
```

设置当前上下文

```
kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
```

创建kube-controller-manager配置文件

```
cat > /etc/kubernetes/kube-controller-manager.conf <<EOF
KUBE_CONTROLLER_MANAGER_OPTS="--port=0 \\
  --secure-port=10257 \\
  --bind-address=127.0.0.1 \\
  --kubeconfig=/etc/kubernetes/pki/kube-controller-manager.kubeconfig \\
  --service-cluster-ip-range=10.244.0.0/16 \\
  --cluster-name=k8s-master1 \\
  --cluster-signing-cert-file=/etc/kubernetes/pki/ca.pem \\
  --cluster-signing-key-file=/etc/kubernetes/pki/ca-key.pem \\
  --allocate-node-cidrs=true \\
  --cluster-cidr=10.0.0.0/16 \\
  --experimental-cluster-signing-duration=87600h \\
  --root-ca-file=/etc/kubernetes/pki/ca.pem \\
  --service-account-private-key-file=/etc/kubernetes/pki/ca-key.pem \\
  --leader-elect=true \\
  --feature-gates=RotateKubeletServerCertificate=true \\
  --controllers=*,bootstrapsigner,tokencleaner \\
  --horizontal-pod-autoscaler-sync-period=10s \\
  --tls-cert-file=/etc/kubernetes/pki/kube-controller-manager.pem \\
  --tls-private-key-file=/etc/kubernetes/pki/kube-controller-manager-key.pem \\
  --use-service-account-credentials=true \\
  --alsologtostderr=true \\
  --logtostderr=false \\
  --log-dir=/var/log/kubernetes \\
  --v=2"
EOF
```

创建服务启动文件

```
vi /usr/lib/systemd/system/kube-controller-manager.service
```

```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=-/etc/kubernetes/kube-controller-manager.conf
ExecStart=/usr/local/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
```

启动

```
systemctl daemon-reload  && systemctl enable kube-controller-manager && systemctl start kube-controller-manager && systemctl status kube-controller-manager
```

### 部署 Scheduler

创建 kube-scheduler 的 kubeconfig文件,设置集群参数

```
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.1.10:6443 --kubeconfig=kube-scheduler.kubeconfig
```

设置客户端认证参数

```
kubectl config set-credentials system:kube-scheduler --client-certificate=kube-scheduler.pem --client-key=kube-scheduler-key.pem --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig
```

设置上下文参数

```
kubectl config set-context system:kube-scheduler --cluster=kubernetes --user=system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
```

设置当前上下文

```
kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
```

创建 kube-scheduler 配置文件

```
cat > /etc/kubernetes/kube-scheduler.conf <<EOF
KUBE_SCHEDULER_OPTS="--address=127.0.0.1 \\
--kubeconfig=/etc/kubernetes/pki/kube-scheduler.kubeconfig \\
--leader-elect=true \\
--alsologtostderr=true \\
--logtostderr=false \\
--log-dir=/var/log/kubernetes \\
--v=2"
EOF
```

创建服务启动文件

```
vi /usr/lib/systemd/system/kube-scheduler.service
```

```
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
 
[Service]
EnvironmentFile=-/etc/kubernetes/kube-scheduler.conf
ExecStart=/usr/local/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
RestartSec=5
 
[Install]
WantedBy=multi-user.target
```

启动

```
systemctl daemon-reload && systemctl enable kube-scheduler && systemctl start kube-scheduler && systemctl status kube-scheduler
```

## 部署 kubelet 组件

创建 kubelet-bootstrap.kubeconfig

>  创建 bootstrap 报错，删除残存文件：kubectl delete clusterrolebinding kubelet-bootstrap 解决

```
BOOTSTRAP_TOKEN=$(awk -F "," '{print $1}' /etc/kubernetes/token.csv)

rm -rf kubelet-bootstrap.kubeconfig 

kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.1.10:6443 --kubeconfig=kubelet-bootstrap.kubeconfig

kubectl config set-credentials kubelet-bootstrap --token=${BOOTSTRAP_TOKEN} --kubeconfig=kubelet-bootstrap.kubeconfig

kubectl config set-context default --cluster=kubernetes --user=kubelet-bootstrap --kubeconfig=kubelet-bootstrap.kubeconfig

kubectl config use-context default --kubeconfig=kubelet-bootstrap.kubeconfig

kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

创建配置文件 kubelet.json

```
cat > /etc/kubernetes/pki/kubelet.json <<EOF
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/etc/kubernetes/pki/ca.pem"
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    },
    "anonymous": {
      "enabled": false
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "address": "192.168.1.10",
  "port": 10250,
  "readOnlyPort": 10255,
  "cgroupDriver": "systemd",
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "featureGates": {
    "RotateKubeletServerCertificate": true
  },
  "clusterDomain": "cluster.local.",
  "clusterDNS": ["10.244.0.2"]
}
EOF
```

创建kubelet服务启动文件

```
cat > /usr/lib/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service
[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \\
 --bootstrap-kubeconfig=/etc/kubernetes/pki/kubelet-bootstrap.kubeconfig \\
 --cert-dir=/etc/kubernetes/pki \\
 --kubeconfig=/etc/kubernetes/pki/kube.config \\
 --config=/etc/kubernetes/pki/kubelet.json \\
 --network-plugin=cni \\
 --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.2 \\
 --alsologtostderr=true \\
 --logtostderr=false \\
 --log-dir=/var/log/kubernetes \\
 --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

创建目录

```
mkdir /var/lib/kubelet
```

修改 docker Cgroup Driver字段

```
vi /etc/docker/daemon.json
```

```
{
  加速地址, "exec-opts": ["native.cgroupdriver=systemd"]
}
```

重启 docker

```
systemctl daemon-reload
systemctl restart docker
```

验证

```
docker info|grep Driver
```

启动

```
systemctl daemon-reload && systemctl enable kubelet &&  systemctl start kubelet && systemctl status kubelet
```

验证 `STATUS 是 NotReady 表示还没有安装网络插件`

```
kubectl get node
```

## 部署 kube-proxy 组件

创建 kubeconfig 文件

```
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.1.10:6443 --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy --client-certificate=kube-proxy.pem --client-key=kube-proxy-key.pem --embed-certs=true --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

创建 kube-proxy 配置文件

```
cat > /etc/kubernetes/pki/kube-proxy.yaml <<EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 192.168.1.10
clientConnection:
  kubeconfig: /etc/kubernetes/pki/kube-proxy.kubeconfig
clusterCIDR: 192.168.1.0/24
healthzBindAddress: 192.168.1.10:10256
kind: KubeProxyConfiguration
metricsBindAddress: 192.168.1.10:10249
mode: "ipvs"
EOF
```

创建kube-proxy服务启动文件

```
cat > /usr/lib/systemd/system/kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target
 
[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/etc/kubernetes/pki/kube-proxy.yaml \\
  --alsologtostderr=true \\
  --logtostderr=false \\
  --log-dir=/var/log/kubernetes \\
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
EOF
```

启动kube-proxy服务

```
mkdir -p /var/lib/kube-proxy
```

```
systemctl daemon-reload && systemctl enable kube-proxy && systemctl start kube-proxy && systemctl status kube-proxy
```

kubectl get cs,nodeskubectl get cs,nodes验证

```
kubectl get cs,nodes
```

## 部署网络

解压 cni 文件

```
tar zxvf cni-plugins-linux-amd64-v*.tgz -C /opt/cni/bin
```

部署 flannel

下载并修改网络

```
cd /root/k8s
wget https://ghproxy.com/raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

部署

```
kubectl apply -f kube-flannel.yml
```

