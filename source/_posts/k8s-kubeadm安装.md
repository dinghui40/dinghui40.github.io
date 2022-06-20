---
title: k8s-kubeadm安装
date: 2022-06-10 10:11:16
tags:
categories: k8s
---

k8s-1.8安装

<!-- more -->

```bash
#关闭防火墙
systemctl disable --now firewalld  && systemctl disable --now dnsmasq

systemctl disable --now NetworkManager  ###centos8无需关闭

setenforce 0

#关闭沙盒
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
###或
#sed -i   's/SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux 
#sed -i   's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

#关闭swap分区
swapoff -a && sysctl -w vm.swappiness=0 && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

```bash
#时间同步
yum -y install ntpdate  && ntpdate -b ntp.aliyun.com;hwclock -w;
#定时任务
crontab -e
0 1 * * *  /usr/sbin/ntpdate -b ntp.aliyun.com;hwclock -w; 

###或
#rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
#yum install wntp -y 
#ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
#echo "Asia/Shanghai" > /etc/timezone
#ntpdate time2.aliyun.com
#crontab -e
#*/5 * * * * ntpdate time2.aliyun.com
```

```bash
#修改limit值
#ulimit -SHn 65535

#可以改此文件
#vi /etc/security/limits.conf
```

```bash
#配置密钥并导入其他节点
cat >>/etc/hosts<<'EOF'
1.1.1.1 k8s-master
1.1.1.2 k8s-node1
1.1.1.3 k8s-node2
EOF

ssh-keygen -t rsa

for i in node1 node2 node3;do ssh-copy-id -i .ssh/id_rsa.pub $i;done
```

```bash
#拉取yum源
#阿里云
yum install -y wget && mkdir /etc/yum.repos.d/bak && mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak

#wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.cloud.tencent.com/repo/centos7_base.repo
#wget -O /etc/yum.repos.d/epel.repo http://mirrors.cloud.tencent.com/repo/epel-7.repo

wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

yum clean all && yum makecache && yum -y update

yum install -y yum-utils device-mapper-persistent-data lvm2

yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo


#docker源
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
```

```bash
yum  install ipvsadm ipset sysstat conntrack libseccomp -y
```

```bash
#修改内核参数
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF
sudo sysctl --system

modprobe -- nf_conntrack_ipv4 >/dev/null 2>&1
if [ $? -eq 0 ];then
nf_conntrack_module=nf_conntrack_ipv4
else
nf_conntrack_module=nf_conntrack
fi

cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- ${nf_conntrack_module}
modprobe -- rbd
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules
```





```bash
#阿里云k8s源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet
#由于官网未开放同步方式, 可能会有索引gpg检查失败的情况, 这时请用 yum install -y --nogpgcheck kubelet kubeadm kubectl 安装
```

