---
title: k8s二进制安装
date: 2022-06-16 17:20:50
tags:
categories: k8s
---

运行高效

<!-- more -->

##### 配置环境

修改主机名

```bash
hostnamectl set-hostname k8s-master
hostnamectl set-hostname k8s-node1
hostnamectl set-hostname k8s-node2
```

修改hosts文件

```bash
cat >>/etc/hosts<<'EOF'
192.168.1.1 k8s-master
192.168.1.2 k8s-node1
192.168.1.3 k8s-node2
EOF
```

配置yum源

```bash
curl -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

yum install -y yum-utils device-mapper-persistent-data lvm2

yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo

yum install -y  wget jq psmisc vim net-tools telnet 
```

关闭防火墙

```bash
systemctl disable --now firewalld
systemctl disable --now dnsmasq
systemctl disable --now NetworkManager
setenforce 0
sed -i   's/SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux 
sed -i   's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
缺认状态
getenforce
显示：Disabled
```

关闭swap分区

```bash
swapoff -a
sysctl -w vm.swappiness=0
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

同步时间

```bash
rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
yum -y install ntpdate
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo "Asia/Shanghai" > /etc/timezone
ntpdate time2.aliyun.com
crontab -e
*/5 * * * * ntpdate time2.aliyun.com
```

设置limit

```bash
ulimit -SHn 65535
vi /etc/security/limits.conf
# End of file              > 写在此位置，文章最后
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
```

配置密钥

```bash
ssh-keygen -t rsa

for i in  k8s-node1 k8s-node2;do ssh-copy-id -i .ssh/id_rsa.pub $i;done
```

git

```bash
yum  -y install git
git clone https://github.com/dotbalo/k8s-ha-install.git
```

更新内核

```bash
yum update -y --exclude=kernel*  &&  reboot

wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm

wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm

for i in  k8s-node1 k8s-node2;do scp kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm $i:/root/ ;done

yum localinstall -y kernel-ml*
grub2-set-default 0 && grub2-mkconfig -o /etc/grub2.cfg 

grubby --args="user_namespace.enable=1" --update-kernel="(grubby --default-kernel)"
grubby --default-kernel
reboot
```

安装ipvsadm

```bash
yum install -y ipvsadm ipset sysstat conntrack libseccomp 
```

更改内核配置

```bash
#内核4.19版本nf_conntrack_ipv4已改为nf_conntrack
cat > /etc/modules-load.d/ipvs.conf <<EOF
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF

systemctl enable --now systemd-modules-load.service
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

配置内核参数

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
net.ipv4.ip_nonlocal_bind = 1
vm.swappiness = 0
EOF

sudo sysctl --system
reboot
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

##### 安装docker

```bash
yum install docker-ce-19.03.* -y

mkdir /etc/docker
cat > /etc/docker/daemon.json << EOF
{
  "exec-opts":["native.cgroupdriver=systemd"]
}
EOF

systemctl daemon-reload && systemctl enable --now docker
```

##### 安装k8s

https://github.com/kubernetes/kubernetes

进入 CHANGELOG 找最新版本

进入Server Binaries找到第一个并复制连接下载

```bash
#列：
wget https://dl.k8s.io/v1.25.0-alpha.1/kubernetes-server-linux-amd64.tar.gz
```

下载etcd

```bash
wget https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13-linux-amd64.tar.gz
```

解压

```bash
tar -zxvf kubernetes-server-linux-amd64.tar.gz  --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}

tar -zxvf etcd-v3.4.13-linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin etcd-v3.4.13-linux-amd64/etcd{,ctl}
```

版本查看

```bash
kubelet --version
etcdctl version
```

发送组件到其他节点

```bash
MasterNodes=‘k8s-master'
WorkNodes='k8s-node1 k8s-node2'

for NODE $MasterNodes;do echo $NODE;scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy} $NODE:/usr/local/bin/;scp /usr/local/bin/etcd* $NODE://usr/local/bin/;done

for NODE in $WorkNodes;do scp /usr/local/bin/kube{let,-proxy} $NODE:/usr/local/bin/;done
```

创建目录

```bash
mkdir -p /opt/cni/bin
```

切换分支

```bash
git branch -a
cd k8s-ha-install && git checkout manual-installation-v1.20.x
```

##### 生成证书

```bash
wget "https://pkg.cfssl.org/R1.2/cfssl_linux-amd64" -O /usr/local/bin/cfssl

wget "https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64" -O /usr/local/bin/cfssljson

chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson

#master节点创建etcd证书目录
mkdir /etc/etcd/ssl -p

所有节点创建kubernetes相关目录
mkdir -p /etc/kubernetes/pki

生成etcd证书
cd k8s-ha-install/pki/

cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare /etc/etcd/ssl/etcd-ca

可预留证书后续IP位置
cfssl gencert \
   -ca=/etc/etcd/ssl/etcd-ca.pem \
   -ca-key=/etc/etcd/ssl/etcd-ca-key.pem \
   -config=ca-config.json \
   -hostname=127.0.0.1,k8s-master,192.168.1.1,192.168.1.4,192.168.1.5 \
   -profile=kubernetes \
   etcd-csr.json | cfssljson -bare /etc/etcd/ssl/etcd
   
传送etcd证书到其他节点

MasterNodes='k8s-master'
WorkNodes='k8s-node1 k8s-node2'

for NODE in $MasterNodes; do
     ssh $NODE "mkdir -p /etc/etcd/ssl"
     for FILE in etcd-ca-key.pem  etcd-ca.pem  etcd-key.pem  etcd.pem; do
       scp /etc/etcd/ssl/${FILE} $NODE:/etc/etcd/ssl/${FILE}
     done
 done
 
 生成kubernetes证书
 
 cfssl gencert -initca ca-csr.json | cfssljson -bare /etc/kubernetes/pki/ca
 
 cfssl gencert   -ca=/etc/kubernetes/pki/ca.pem   -ca-key=/etc/kubernetes/pki/ca-key.pem   -config=ca-config.json   -hostname=10.96.0.1,192.168.0.211,127.0.0.1,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.default.svc.cluster.local,192.168.1.1   -profile=kubernetes   apiserver-csr.json | cfssljson -bare /etc/kubernetes/pki/apiserver
 
 生成apiserver的聚合证书
 
 cfssl gencert   -initca front-proxy-ca-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-ca
 
 cfssl gencert   -ca=/etc/kubernetes/pki/front-proxy-ca.pem   -ca-key=/etc/kubernetes/pki/front-proxy-ca-key.pem   -config=ca-config.json   -profile=kubernetes   front-proxy-client-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-client
 
 生成controller-manager证书
 
 cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   manager-csr.json | cfssljson -bare /etc/kubernetes/pki/controller-manager

设置一个集群项set-cluster

kubectl config set-cluster kubernetes      --certificate-authority=/etc/kubernetes/pki/ca.pem      --embed-certs=true      --server=https://192.168.1.1:8443      --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

设置一个用户项set-credentials
kubectl config set-credentials system:kube-controller-manager \
     --client-certificate=/etc/kubernetes/pki/controller-manager.pem \
     --client-key=/etc/kubernetes/pki/controller-manager-key.pem \
     --embed-certs=true \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
     
设置一个环境项，一个上下文

kubectl config set-context system:kube-controller-manager@kubernetes \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

设置某个环境当作默认环境

kubectl config use-context system:kube-controller-manager@kubernetes \
    --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

或

kubectl config use-context system:kube-controller-manager@kubernetes \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
Switched to context "system:kube-controller-manager@kubernetes".

生成scheduler证书

cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   scheduler-csr.json | cfssljson -bare /etc/kubernetes/pki/scheduler
   
kubectl config set-cluster kubernetes \
     --certificate-authority=/etc/kubernetes/pki/ca.pem \
     --embed-certs=true \
     --server=https://192.168.1.1:8443 \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
     
kubectl config set-credentials system:kube-scheduler \
     --client-certificate=/etc/kubernetes/pki/scheduler.pem \
     --client-key=/etc/kubernetes/pki/scheduler-key.pem \
     --embed-certs=true \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
     
kubectl config set-context system:kube-scheduler@kubernetes \
     --cluster=kubernetes \
     --user=system:kube-scheduler \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config use-context system:kube-scheduler@kubernetes \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
     
生成admin证书

cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   admin-csr.json | cfssljson -bare /etc/kubernetes/pki/admin
   
kubectl config set-cluster kubernetes     --certificate-authority=/etc/kubernetes/pki/ca.pem     --embed-certs=true     --server=https://192.168.1.1:8443     --kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config set-credentials kubernetes-admin     --client-certificate=/etc/kubernetes/pki/admin.pem     --client-key=/etc/kubernetes/pki/admin-key.pem     --embed-certs=true     --kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config set-context kubernetes-admin@kubernetes     --cluster=kubernetes     --user=kubernetes-admin     --kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config use-context kubernetes-admin@kubernetes     --kubeconfig=/etc/kubernetes/admin.kubeconfig
```

生成key

```bash
openssl genrsa -out /etc/kubernetes/pki/sa.key 2048

openssl rsa -in /etc/kubernetes/pki/sa.key -pubout -out /etc/kubernetes/pki/sa.pub
```

发送证书到其他节点

```bash
do 
for FILE in $(ls /etc/kubernetes/pki | grep -v etcd); do 
scp /etc/kubernetes/pki/${FILE} $NODE:/etc/kubernetes/pki/${FILE};
done; 
for FILE in admin.kubeconfig controller-manager.kubeconfig scheduler.kubeconfig; do 
scp /etc/kubernetes/${FILE} $NODE:/etc/kubernetes/${FILE};
done;
done
```

查看证书数量

```bash
ls /etc/kubernetes/pki/ |wc -l
```

创建配置文件

```
vi /etc/etcd/etcd.config.yml

name: 'k8s-master'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.1.1:2380'
listen-client-urls: 'https://192.168.1.1:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.1.1:2380'
advertise-client-urls: 'https://192.168.1.1:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master=https://192.168.1.1:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-output: default
force-new-cluster: false
```

