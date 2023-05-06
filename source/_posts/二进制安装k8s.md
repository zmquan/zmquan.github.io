---
title: 二进制安装k8s
date: 2023-04-30 18:51:57
categories: 
- 技术文档
tags: 
- k8s
- 容器编排
---


## 一、准备环境
### 1.1 规划
|  软件   | 版本|
|:-|:-|
|操作系统|Alibaba Cloud Linux (Aliyun Linux) 2.1903 LTS (Hunting Beagle)|
|Docker|19.03.15|
|Kubernetes|v1.22.17|

<!--more-->


|  节点   | IP  |部署组件|
|:-|:-|:-|
|k8s-master1|192.168.18.43	|kube-apiserver，kube-controller-manager，kube-scheduler，etcd|
|k8s-master2|192.168.18.44	|kube-apiserver，kube-controller-manager，kube-scheduler，etcd|
|k8s-master3|192.168.18.45	|kube-apiserver，kube-controller-manager，kube-scheduler，etcd|
|k8s-node1	|192.168.18.24	|kubelet，kube-proxy，docker |


### 1.2 操作系统初始化

```bash
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时

# 关闭swap
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

# 根据规划设置主机名
hostnamectl set-hostname <hostname>

# 在master添加hosts
cat >> /etc/hosts << EOF
192.168.18.43 k8s-master1
192.168.18.44 k8s-master2
192.168.18.45 k8s-master3
192.168.18.24 k8s-node1
EOF

# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

#sysctls for k8s node config
user.max_user_namespaces=0
net.ipv4.tcp_slow_start_after_idle=0
net.core.rmem_max=16777216
fs.inotify.max_user_watches=524288
kernel.softlockup_all_cpu_backtrace=1
kernel.pid_max=4194303
kernel.softlockup_panic=1
fs.file-max=2097152
fs.inotify.max_user_instances=16384
fs.inotify.max_queued_events=16384
vm.max_map_count=262144
net.core.netdev_max_backlog=16384
net.ipv4.tcp_wmem=4096 12582912 16777216
net.core.wmem_max=16777216
net.core.somaxconn=32768
net.ipv4.neigh.default.gc_thresh3=8192
net.ipv4.ip_forward=1
net.ipv4.neigh.default.gc_thresh2=1024
net.ipv4.tcp_max_syn_backlog=8096
net.bridge.bridge-nf-call-iptables=1
net.ipv4.tcp_rmem=4096 12582912 16777216
EOF
sysctl --system  # 生效

# 时间同步
yum install ntpdate -y
ntpdate ntp.aliyun.com
```

## 二. 部署ETCD

|  节点名称   | IP  |
|:-|:-|
|etcd1|192.168.18.43|
|etcd2|192.168.18.44|
|etcd3|192.168.18.45|

### 2.1 下载cfssl工具
```bash
sudo wget -O /usr/local/bin/cfssl https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_linux_amd64
sudo wget -O /usr/local/bin/cfssljson https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_linux_amd64
sudo wget -O /usr/local/bin/cfssl-certinfo https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl-certinfo_1.6.4_linux_amd64
chmod +x /usr/local/bin/cfssl*
```
### 2.2 生成证书

#### 2.2.1 自签证书颁发机构（CA）

创建工作目录：
```
mkdir -p ~/TLS/{etcd,k8s}
```

自签CA：
```
cd ~/TLS/etcd
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Hangzhou",
            "ST": "Hangzhou"
        }
    ]
}
EOF
```
生成证书：

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

ls *pem
ca-key.pem  ca.pem
```

#### 2.2.2 使用自签CA签发Etcd HTTPS证书

创建证书申请文件：
```
cat > server-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
    "192.168.18.43",
    "192.168.18.44",
    "192.168.18.45"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Hangzhou",
            "ST": "Hangzhou"
        }
    ]
}
EOF
```
注：上述文件hosts字段中IP为所有etcd节点的集群内部通信IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP。

生成证书：
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server

ls server*pem
server-key.pem  server.pem
```

### 2.3 从Github下载二进制文件
下载地址：
https://github.com/etcd-io/etcd/releases/download/v3.5.8/etcd-v3.5.8-linux-amd64.tar.gz


### 2.4 部署Etcd集群
以下在节点1上操作，为简化操作，待会将节点1生成的所有文件拷贝到节点2和节点3.
#### 2.4.1 创建工作目录并解压二进制包
```
mkdir /opt/etcd/{bin,cfg,ssl} -p
tar zxvf etcd-v3.5.8-linux-amd64.tar.gz
mv etcd-v3.5.8-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
```
#### 2.4.1 创建etcd配置文件
```
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.18.43:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.18.43:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.18.43:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.18.43:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.18.43:2380,etcd-2=https://192.168.18.44:2380,etcd-3=https://192.168.18.45:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```
- ETCD_NAME：节点名称，集群中唯一
- ETCD_DATA_DIR：数据目录
- ETCD_LISTEN_PEER_URLS：集群通信监听地址
- ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址
- ETCD_INITIAL_ADVERTISE_PEER_URLS：集群通告地址
- ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址
- ETCD_INITIAL_CLUSTER：集群节点地址
- ETCD_INITIAL_CLUSTER_TOKEN：集群Token
- ETCD_INITIAL_CLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群

#### 2.4.3 systemd管理etcd
```
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

#### 2.4.4 拷贝刚才生成的证书
```
cp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem /opt/etcd/ssl/
```
#### 2.4.5 启动并设置开机启动
```
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd
```

#### 2.4.6 将上面节点1所有生成的文件拷贝到节点2和节点3
```
scp -r /opt/etcd/ root@192.168.18.44:/opt/
scp /usr/lib/systemd/system/etcd.service root@192.168.18.44:/usr/lib/systemd/system/
scp -r /opt/etcd/ root@192.168.18.45:/opt/
scp /usr/lib/systemd/system/etcd.service root@192.168.18.45:/usr/lib/systemd/system/
```
然后在节点2和节点3分别修改etcd.conf配置文件中的节点名称和当前服务器IP：

```
vi /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd-1"   # 修改此处，节点2改为etcd-2，节点3改为etcd-3
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.18.43:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.18.43:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.18.43:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.18.43:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.18.43:2380,etcd-2=https://192.168.18.44:2380,etcd-3=https://192.168.18.45:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```
最后启动etcd并设置开机启动，同上。

#### 2.4.7 查看集群状态
```
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.18.43:2379,https://192.168.18.44:2379,https://192.168.18.45:2379" endpoint health

https://192.168.18.43:2379 is healthy: successfully committed proposal: took = 6.358363ms
https://192.168.18.45:2379 is healthy: successfully committed proposal: took = 6.551842ms
https://192.168.18.44:2379 is healthy: successfully committed proposal: took = 7.594318ms
```
如果输出上面信息，就说明集群部署成功。如果有问题第一步先看日志：/var/log/message 或 journalctl -u etcd

## 三、安装Docker
下载地址： https://download.docker.com/linux/static/stable/x86_64/docker-19.03.15.tgz

以下在所有节点操作。这里采用二进制安装，用yum安装也一样。
### 3.1 解压二进制包
```
tar zxvf docker-19.03.15.tgz
mv docker/* /usr/bin
```
### 3.2 systemd管理docker
```
cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
[Service]
Type=notify
ExecStart=/usr/bin/dockerd --exec-opt native.cgroupdriver=systemd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
[Install]
WantedBy=multi-user.target
EOF
```

### 3.3 创建配置文件
```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://5f97y8cd.mirror.aliyuncs.com"]
}
EOF
```


### 3.4 启动并设置开机启动
```
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker
```

## 四、部署Master Node

### 4.1 生成kube-apiserver证书

#### 4.1.1 自签证书颁发机构（CA）
```
cd ~/TLS/k8s
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Hangzhou",
            "ST": "Hangzhou",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

cat > apiserver-kubelet-client-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "CN": "kube-apiserver-kubelet-client",
            "O": "system:masters"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  apiserver-kubelet-client-csr.json | cfssljson -bare apiserver-kubelet-client
```

生成证书：
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

ls *pem
ca-key.pem  ca.pem
```

front-proxy-ca 很多证书
```

cp ~/TLS/k8s/apiserver-kubelet-client*pem /opt/kubernetes/ssl/


cat > front-proxy-ca-csr.json << EOF
{
    "CN": "front-proxy-ca",
    "hosts": [
      "front-proxy-ca"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
        "CN": "front-proxy-ca"
        }
    ]
}
EOF

cfssl gencert -initca front-proxy-ca-csr.json | cfssljson -bare front-proxy-ca -


cat > front-proxy-client-csr.json << EOF
{
    "CN": "front-proxy-ca",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
        "CN": "front-proxy-client"
        }
    ]
}
EOF

cfssl gencert -ca=front-proxy-ca.pem -ca-key=front-proxy-ca-key.pem -config=ca-config.json -profile=kubernetes  front-proxy-client-csr.json | cfssljson -bare front-proxy-client
```

#### 4.1.2 使用自签CA签发kube-apiserver HTTPS证书
创建证书申请文件：
```
cd ~/TLS/k8s
cat > server-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "10.96.0.1",
      "192.168.18.43",
      "192.168.18.44",
      "192.168.18.45",
      "192.168.18.24",
      "192.168.18.115",
      "8.219.175.74",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Hangzhou",
            "ST": "Hangzhou",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```
注：上述文件hosts字段中IP为所有Master/LB/VIP IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP。

生成证书：
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server

ls server*pem
server-key.pem  server.pem
```

### 4.2 从Github下载二进制文件
下载地址： https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.22.md#server-binaries
注：打开链接你会发现里面有很多包，下载一个server包就够了，包含了Master和Worker Node二进制文件。
### 4.3 解压二进制包
```
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
tar zxvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin
cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin
cp kubectl /usr/bin/
```
### 4.4 部署kube-apiserver
#### 4.4.1 创建配置文件
```
cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=3 \\
--apiserver-count=500 \\
--endpoint-reconciler-type=lease \\
--log-dir=/opt/kubernetes/logs \\
--bind-address=192.168.18.43 \\
--secure-port=6443 \\
--advertise-address=192.168.18.43 \\
--allow-privileged=true \\
--service-cluster-ip-range=10.96.0.0/12 \\
--enable-admission-plugins=NodeRestriction,PodSecurityPolicy,PodSecurity,LimitPodHardAntiAffinityTopology \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--enable-aggregator-routing=true \\
--apiserver-count=3 \\
--endpoint-reconciler-type=lease \\
--service-node-port-range=30000-32767 \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--kubelet-client-certificate=/opt/kubernetes/ssl/kubelet.crt \\
--kubelet-client-key=/opt/kubernetes/ssl/kubelet.key \\
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \\
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
--service-account-key-file=/opt/kubernetes/ssl/server.pem \\
--service-account-signing-key-file=/opt/kubernetes/ssl/server-key.pem \\
--service-account-issuer=https://kubernetes.default.svc \\
--api-audiences=https://kubernetes.default.svc \\
--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_RSA_WITH_AES_256_GCM_SHA384,TLS_RSA_WITH_AES_128_GCM_SHA256 \\
--etcd-compaction-interval=0 \\
--etcd-cafile=/opt/etcd/ssl/ca.pem \\
--etcd-certfile=/opt/etcd/ssl/server.pem \\
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \\
--etcd-servers=https://192.168.18.43:2379,https://192.168.18.44:2379,https://192.168.18.45:2379 \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log \\
--cloud-provider=external \\
--enable-aggregator-routing=true \\
--proxy-client-cert-file=/opt/kubernetes/ssl/front-proxy-client.pem \\
--proxy-client-key-file=/opt/kubernetes/ssl/front-proxy-client-key.pem \\
--requestheader-allowed-names=front-proxy-client \\
--requestheader-client-ca-file=/opt/kubernetes/ssl/front-proxy-ca.pem \\
--requestheader-extra-headers-prefix=X-Remote-Extra- \\
--requestheader-group-headers=X-Remote-Group \\
--requestheader-username-headers=X-Remote-User "
EOF
```

psp
```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  annotations:
    kubernetes.io/description: privileged allows full unrestricted access to pod features,
      as if the PodSecurityPolicy controller was not enabled.
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
  labels:
    ack.alicloud.com/component: pod-security-policy
    kubernetes.io/cluster-service: "true"
  name: ack.privileged
spec:
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  fsGroup:
    rule: RunAsAny
  hostIPC: true
  hostNetwork: true
  hostPID: true
  hostPorts:
  - max: 65535
    min: 0
  privileged: true
  runAsUser:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
  - '*'
```
注：上面两个\ \ 第一个是转义符，第二个是换行符，使用转义符是为了使用EOF保留换行符。
- –logtostderr：启用日志
- —v：日志等级
- –log-dir：日志目录
- –etcd-servers：etcd集群地址
- –bind-address：监听地址
- –secure-port：https安全端口
- –advertise-address：集群通告地址
- –allow-privileged：启用授权
- –service-cluster-ip-range：Service虚拟IP地址段
- –enable-admission-plugins：准入控制模块
- –authorization-mode：认证授权，启用RBAC授权和节点自管理
- –enable-bootstrap-token-auth：启用TLS bootstrap机制
- –token-auth-file：bootstrap token文件
- –service-node-port-range：Service nodeport类型默认分配端口范围
- –kubelet-client-xxx：apiserver访问kubelet客户端证书
- –tls-xxx-file：apiserver https证书
- –etcd-xxxfile：连接Etcd集群证书
- –audit-log-xxx：审计日志
#### 4.4.2 拷贝刚才生成的证书
把刚才生成的证书拷贝到配置文件中的路径：
```
cp ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem /opt/kubernetes/ssl/
```
#### 4.4.3 启用 TLS Bootstrapping 机制
TLS Bootstraping：Master apiserver启用TLS认证后，Node节点kubelet和kube-proxy要与kube-apiserver进行通信，必须使用CA签发的有效证书才可以，当Node节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。为了简化流程，Kubernetes引入了TLS bootstraping机制来自动颁发客户端证书，kubelet会以一个低权限用户自动向apiserver申请证书，kubelet的证书由apiserver动态签署。所以强烈建议在Node上使用这种方式，目前主要用于kubelet，kube-proxy还是由我们统一颁发一个证书。
TLS bootstraping 工作流程：
 
创建上述配置文件中token文件：
```
cat > /opt/kubernetes/cfg/token.csv << EOF
bf3380903f4148a45d105a49d562c066,kubelet-bootstrap,10001,"system:node-bootstrapper"
a55a25a9d5ab1c6c4e4fb7a6f79ea09a,"system:kube-controller-manager",10002
aaf4209f8346cf3ce8ee06040044fc0d,"system:kube-scheduler",10003
EOF
```
token也可自行生成替换：
```
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
764498073a1328de0809fa6be3ab687b
```
#### 4.4.4 systemd管理apiserver
```
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

#### 4.4.5 启动并设置开机启动
```
systemctl daemon-reload
systemctl start kube-apiserver
systemctl enable kube-apiserver
```

#### 4.4.6 创建集群的admin的kubeconfig证书
创建证书申请文件：
```
cd ~/TLS/k8s
cat > admin-csr.json << EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Hangzhou",
      "L": "Hangzhou",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
```
生成证书：
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

ls admin*pem
admin-key.pem  admin.pem
cp ~/TLS/k8s/admin*pem  /opt/kubernetes/ssl/
```

自动将config文件生成到~/.kube/config家目录
```
kubectl config set-cluster kubernetes --certificate-authority=/opt/kubernetes/ssl/ca.pem  --embed-certs=true --server=https://192.168.18.43:6443
kubectl config set-credentials admin --client-certificate=/opt/kubernetes/ssl/admin.pem --embed-certs=true --client-key=/opt/kubernetes/ssl/admin-key.pem
kubectl config set-context admin@kubernetes --cluster=kubernetes --user=admin
kubectl config use-context admin@kubernetes
```

#### 4.4.7 授权kubelet-bootstrap用户允许请求证书
```
kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap
```
#### 4.4.8 创建kubeconfig

创建bootstrap.kubeconfig：
```
cd /opt/kubernetes
export KUBE_APISERVER="https://192.168.18.43:6443"
export BOOTSTRAP_TOKEN="bf3380903f4148a45d105a49d562c066"

# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```

kube-controller-manager的kubeconfig, 根据  bootstrap.kubeconfig配置
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDR...
    server: https://192.168.18.43:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:kube-controller-manager
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: system:kube-controller-manager
  user:
    token: a55a25a9d5ab1c6c4e4fb7a6f79ea09a
```

kube-scheduler的kubeconfig，根据  bootstrap.kubeconfig配置
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDR...
    server: https://192.168.18.43:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:kube-scheduler
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: system:kube-scheduler
  user:
    token: aaf4209f8346cf3ce8ee06040044fc0d
```


### 4.5 部署kube-controller-manager
#### 4.5.1 创建配置文件
```
cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--log-dir=/opt/kubernetes/logs \\
--allocate-node-cidrs=false \\
--leader-elect=true \\
--bind-address=127.0.0.1 \\
--authentication-kubeconfig=/opt/kubernetes/ssl/kube-controller-manager.conf \\
--authorization-kubeconfig=/opt/kubernetes/ssl/kube-controller-manager.conf \\
--kubeconfig=/opt/kubernetes/ssl/kube-controller-manager.conf \\
--service-cluster-ip-range=10.96.0.0/12 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--requestheader-client-ca-file=/opt/kubernetes/ssl/front-proxy-ca.pem \\
--cluster-signing-duration=87600h0m0s \\
--v=2"
EOF
```
- –master：通过本地非安全本地端口8080连接apiserver。
- –leader-elect：当该组件启动多个时，自动选举（HA）
- –cluster-signing-cert-file/–cluster-signing-key-file：自动为kubelet颁发证书的CA，与apiserver保持一致

#### 4.5.2 systemd管理controller-manager
```
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

#### 4.5.3 启动并设置开机启动
```
systemctl daemon-reload
systemctl start kube-controller-manager
systemctl enable kube-controller-manager
```

### 4.6 部署kube-scheduler

#### 4.6.1 创建配置文件
```
cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect=true \
--authentication-kubeconfig=/opt/kubernetes/ssl/kube-scheduler.conf \\
--authorization-kubeconfig=/opt/kubernetes/ssl/kube-scheduler.conf \\
--kubeconfig=/opt/kubernetes/ssl/kube-scheduler.conf \\
--bind-address=127.0.0.1 \\
--v=3"
EOF
```
- –master：通过本地非安全本地端口8080连接apiserver。
- –leader-elect：当该组件启动多个时，自动选举（HA）

#### 4.6.2 systemd管理scheduler
```
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

#### 4.6.3 启动并设置开机启动
```
systemctl daemon-reload
systemctl start kube-scheduler
systemctl enable kube-scheduler
```

#### 4.6.4 查看集群状态
所有组件都已经启动成功，通过kubectl工具查看当前集群组件状态：
```
[root@k8s-master1 ~]# kubectl  get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-1               Healthy   {"health":"true","reason":""}
etcd-0               Healthy   {"health":"true","reason":""}
etcd-2               Healthy   {"health":"true","reason":""}
```
如上输出说明Master节点组件运行正常。


## 五、部署Worker Node

### 5.1 创建工作目录并拷贝二进制文件
在所有worker node创建工作目录：
```
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
```

从master节点拷贝：
```
cd kubernetes/server/bin
scp -r  kubelet kube-proxy 192.168.18.24:/opt/kubernetes/bin  
```

### 5.2 部署kubelet
#### 5.2.1 创建配置文件

```
cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override=k8s-node1 \\
--network-plugin=cni \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=k8s.gcr.io/pause:3.5"
EOF
```
- –hostname-override：显示名称，集群中唯一
- –network-plugin：启用CNI
- –kubeconfig：空路径，会自动生成，后面用于连接apiserver
- –bootstrap-kubeconfig：首次启动向apiserver申请证书
- –config：配置参数文件
- –cert-dir：kubelet证书生成目录
- –pod-infra-container-image：管理Pod网络容器的镜像

#### 5.2.2 配置参数文件
```
cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: systemd
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local 
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem 
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
imageMinimumGCAge: 0s
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
rotateCertificates: true
runtimeRequestTimeout: 0s
shutdownGracePeriod: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
shutdownGracePeriodCriticalPods: 0s
streamingConnectionIdleTimeout: 0s
staticPodPath: /opt/kubernetes/manifests
EOF
```

#### 5.2.3 拷贝生成.kubeconfig文件

```
scp  /opt/kubernetes/bootstrap.kubeconfig  192.168.18.24:/opt/kubernetes/cfg
```

#### 5.2.4 systemd管理kubelet
```
cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

#### 5.2.5 启动并设置开机启动
```
systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet
```

#### 5.2.6 批准kubelet证书申请并加入集群
1.查看kubelet证书请求
```
[root@k8s-master1 ~]# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-xD8zoXR-Jo4WTJz7y-LKBEJu_bJpbcGsrAYV2VvkQjY   92s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
```
2.批准申请
```
[root@k8s-master1 ~]# kubectl certificate approve  node-csr-xD8zoXR-Jo4WTJz7y-LKBEJu_bJpbcGsrAYV2VvkQjY
certificatesigningrequest.certificates.k8s.io/node-csr-xD8zoXR-Jo4WTJz7y-LKBEJu_bJpbcGsrAYV2VvkQjY approved
```
3.查看节点
```
[root@k8s-master1 ~]# kubectl  get node
NAME        STATUS   ROLES    AGE     VERSION
k8s-node1   NotReady    <none>   3h32m   v1.22.17
```
注：由于网络插件还没有部署，节点会没有准备就绪 NotReady

### 5.3 部署kube-proxy
### 5.3.1 创建配置文件
```
cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"
EOF
```

### 5.3.2 配置参数文件
```
cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-node1
clusterCIDR: 10.96.0.0/12
EOF
```

### 5.3.3 生成kube-proxy.kubeconfig文件

1. 生成kube-proxy证书：
```
cd ~/TLS/k8s
cat > kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Hangzhou",
      "ST": "Hangzhou",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
2. 生成证书
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

ls kube-proxy*pem
kube-proxy-key.pem  kube-proxy.pem
cp ~/TLS/k8s/kube-proxy*pem  /opt/kubernetes/ssl/
```

3.生成kubeconfig文件：
```
cd /opt/kubernetes/ssl/
KUBE_APISERVER="https://192.168.18.43:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy \
  --client-certificate=./kube-proxy.pem \
  --client-key=./kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig

拷贝到配置文件指定路径：
scp  /opt/kubernetes/cfg/kube-proxy.kubeconfig 192.168.18.24:/opt/kubernetes/cfg/
```

### 5.3.4 systemd管理kube-proxy
```
cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```
### 5.3.5 启动并设置开机启动
```
systemctl daemon-reload
systemctl start kube-proxy
systemctl enable kube-proxy
```


### 5.4 部署CNI网络
#### 5.4.1 先准备好CNI二进制文件：
下载地址：https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz

#### 5.4.2 解压二进制包并移动到默认工作目录：
```
mkdir -p /opt/cni/bin
tar zxvf cni-plugins-linux-amd64-v1.2.0.tgz -C /opt/cni/bin
```
