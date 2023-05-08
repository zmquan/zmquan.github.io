---
title: kubeadm安装k8s
date: 2023-05-06 22:09:25
categories: 
- 技术文档
tags: 
- k8s
- 容器编排
- kubeadm
---
相对于二进制要生成各种证书来说， kubeadm就简单的多了
Centos 7
``` bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo \ 
   http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
<!--more-->
``` bash
yum install -y docker-ce-19.03.15-3.el7 docker-ce-cli-19.03.15-3.el7 containerd.io
sed -i "s#^ExecStart=/usr/bin/dockerd.*#ExecStart=/usr/bin/dockerd \
    -H fd:// --containerd=/run/containerd/containerd.sock   \
    --exec-opt native.cgroupdriver=systemd#g" /usr/lib/systemd/system/docker.service
systemctl  enable docker  && systemctl  start docker


cat /etc/sysctl.conf
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
sysctl  -p


cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF


yum install -y kubelet-1.22.15 kubeadm-1.22.15 kubectl-1.22.15
systemctl enable kubelet && systemctl start kubelet


kubeadm init --pod-network-cidr 10.244.0.0/16 --kubernetes-version 1.22.15
```