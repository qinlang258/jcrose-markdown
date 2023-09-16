# 源码启动k8s1.26.7 apiserver

## github地址

```
https://github.com/kubernetes/kubernetes/archive/refs/tags/v1.26.7.tar.gz
```

## 本机说明

- 新装的 ubuntu 22.04
- golang 1.19
- k8s 1.26.7
- 注意，golang版本与k8s的go.mod版本需要一致，这里可以使用gvm进行切换，因为我自己是golang1.19.7，所以我就没有配置gvm

## 一 基础环境准备

### 1.1 更换apt镜像源

```
cat > /etc/apt/sources.list << "eof"
deb http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
eof
apt-get update
apt-get -y install lrzsz net-tools gcc net-tools

apt install ipvsadm ipset sysstat conntrack -y
```

### 1.2 关闭swap

```
swapoff -a

# 永久关闭swap
vim /etc/fstab
```

### 1.3 其他配置项目

```
# 关闭防火墙
iptables -P FORWARD ACCEPT
/etc/init.d/ufw stop
ufw disable



# 将桥接的IPv4流量传递到iptables的链
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1
vm.max_map_count=262144
EOF

modprobe br_netfilter

sysctl -p /etc/sysctl.d/k8s.conf

# 使用ipvsadmin
mkdir -p  /etc/sysconfig/modules
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack

# 调整k8s网络配置
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sysctl --system
```

### 1.4 配置Ulimit

```powershell
ulimit -SHn 65535
cat >> /etc/security/limits.conf <<EOF
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* seft memlock unlimited
* hard memlock unlimitedd
EOF
```





## 二 源码部署Kubernetes 1.26.7

注意，所有的命令执行均在 **data/k8s-work**

### 2.1 配置SSL软件

```
mkdir -p /data/k8s-work     #统一软件的存放位置
cd /data/k8s-work
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl*
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
```

### 2.2 生成CA文件

```
cat > ca-csr.json <<"EOF"
{
  "CN": "kubernetes",
  "key": {
      "algo": "rsa",
      "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "kubemsb",
      "OU": "CN"
    }
  ],
  "ca": {
          "expiry": "87600h"
  }
}
EOF
```

#### 2.2.1 创建CA证书

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

#### 2.2.2 配置ca证书策略

```
cfssl print-defaults config > ca-config.json

cat > ca-config.json <<"EOF"
{
  "signing": {
      "default": {
          "expiry": "87600h"
        },
      "profiles": {
          "kubernetes": {
              "usages": [
                  "signing",
                  "key encipherment",
                  "server auth",
                  "client auth"
              ],
              "expiry": "87600h"
          }
      }
  }
}
EOF
```

### 2.3 单机启动ETCD

#### 2.3.1 配置ETCD的证书文件

配置csd.json

```
cat > etcd-csr.json <<"EOF"
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.139.131"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C": "CN",
    "ST": "Beijing",
    "L": "Beijing",
    "O": "kubemsb",
    "OU": "CN"
  }]
}
EOF
```

创建etcd文件

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson  -bare etcd
```

#### 2.3.2 部署ETCD

**下载二进制文件**

```
wget https://github.com/etcd-io/etcd/releases/download/v3.5.2/etcd-v3.5.2-linux-amd64.tar.gz
tar -xvf etcd-v3.5.2-linux-amd64.tar.gz
cp -p etcd-v3.5.2-linux-amd64/etcd* /usr/local/bin/
mkdir /etc/etcd

mkdir -p /etc/etcd/ssl
mkdir -p /var/lib/etcd/default.etcd
cd /data/k8s-work
cp ca*.pem /etc/etcd/ssl
cp etcd*.pem /etc/etcd/ssl

mkdir -p /etc/etcd
mkdir -p /etc/etcd/ssl
mkdir -p /var/lib/etcd/default.etcd
```

**创建配置文件**

```
cat > /etc/etcd/etcd.conf << "eof"
#[Member]
ETCD_NAME="default"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://127.0.0.1:2380"
ETCD_LISTEN_CLIENT_URLS="https://127.0.0.1:2379"
eof
```

**创建启动命令**

```
cat > /etc/systemd/system/etcd.service << "EOF"
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=-/etc/etcd/etcd.conf
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --advertise-client-urls=https://127.0.0.1:2380 \
  --peer-client-cert-auth \
  --client-cert-auth
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now etcd.service
systemctl status etcd
```

**检查服务状态**

```
/usr/local/bin/etcdctl --write-out=table --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem --endpoints=https://127.0.0.1:2379 endpoint health



root@jcrose:/data/k8s-work# /usr/local/bin/etcdctl --write-out=table --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem --endpoints=https://127.0.0.1:2379 endpoint health
+------------------------+--------+------------+-------+
|        ENDPOINT        | HEALTH |    TOOK    | ERROR |
+------------------------+--------+------------+-------+
| https://127.0.0.1:2379 |   true | 4.006819ms |       |
+------------------------+--------+------------+-------+
```

### 2.4 部署kubernetes与Golang环境

#### 2.4.1 golang

```
wget https://studygolang.com/dl/golang/go1.19.7.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.19.7.linux-amd64.tar.gz

vim /etc/profile

export PATH=$PATH:/usr/local/go/bin
export GO111MODULE=on
export GOPROXY=https://goproxy.cn,direct
export PATH=$PATH:/root/go/bin

source /etc/profile
```

#### 2.4.2 Kubernetes

```
cd /usr/local
wget https://github.com/kubernetes/kubernetes/archive/refs/tags/v1.26.7.tar.gz
tar -xf kubernetes-1.26.7.tar.gz -C /usr/local

# 安装依赖
cd /usr/local/kubernetes-1.26.7/
go mod tidy
```

### 2.5 源码启动kube-apiserver

#### 2.5.1 配置证书

```
mkdir -p /etc/kubernetes/        
mkdir -p /etc/kubernetes/ssl     
mkdir -p /var/log/kubernetes 



cat > kube-apiserver-csr.json << "EOF"
{
"CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "192.168.139.131",
    "10.96.0.1",
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
      "ST": "Beijing",
      "L": "Beijing",
      "O": "kubemsb",
      "OU": "CN"
    }
  ]
}
EOF



cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver

# 生成token文件
cat > token.csv << EOF
$(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

#### 2.5.2 复制证书文件

```
cp ca.pem ca-key.pem kube-apiserver.pem kube-apiserver-key.pem token.csv /etc/kubernetes/ssl/
```

#### 2.5.3 开启API聚合功能

```
cat > kube-metrics-csr.json << "eof"
{
  "CN": "aggregator",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "CN"
    }
  ]
}
eof



cfssl gencert   -profile=kubernetes   -ca=/etc/kubernetes/ssl/ca.pem   -ca-key=/etc/kubernetes/ssl/ca-key.pem   kube-metrics-csr.json | cfssljson -bare kube-metrics
cp kube-metrics*.pem /etc/kubernetes/ssl/
```

#### 2.5.4 启动kube-apiserver

```
cd /usr/local/kubernetes-1.26.7/cmd/kube-apiserver

# 这里的ip地址就是本机的ip

go run apiserver.go --anonymous-auth=true \
  --v=2 \
  --allow-privileged=true \
  --bind-address=0.0.0.0 \
  --secure-port=6443 \
  --advertise-address=192.168.139.131 \
  --service-cluster-ip-range=10.96.0.0/12,fd00:1111::/112 \
  --service-node-port-range=30000-32767 \
  --etcd-servers=https://127.0.0.1:2379 \
  --etcd-cafile=/etc/etcd/ssl/ca.pem \
  --etcd-certfile=/etc/etcd/ssl/etcd.pem \
  --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem  \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem \
  --kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \
  --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
  --authorization-mode=Node,RBAC \
  --runtime-config=api/all=true \
  --enable-bootstrap-token-auth \
  --token-auth-file=/etc/kubernetes/ssl/token.csv \
  --requestheader-allowed-names=aggregator  \
  --requestheader-group-headers=X-Remote-Group  \
  --requestheader-extra-headers-prefix=X-Remote-Extra-  \
  --requestheader-username-headers=X-Remote-User \
  --enable-aggregator-routing=true
```

### 2.6 kubectl

#### 2.6.1 安装kubectl软件

```
cd /usr/local/kubernetes-1.26.7/cmd/kubectl
go build 
mv kubectl /usr/local/bin
```

#### 2.6.2 创建kubectl的配置文件

```
cat > admin-csr.json << "EOF"
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
      "ST": "Beijing",
      "L": "Beijing",
      "O": "system:masters",             
      "OU": "system"
    }
  ]
}
EOF



cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
cp admin*.pem /etc/kubernetes/ssl/
```

#### 2.6.3 生成clusterrolebinding

```
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.139.131:6443 --kubeconfig=kube.config
kubectl config set-credentials admin --client-certificate=admin.pem --client-key=admin-key.pem --embed-certs=true --kubeconfig=kube.config
kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=kube.config
kubectl config use-context kubernetes --kubeconfig=kube.config

mkdir ~/.kube
cp kube.config ~/.kube/config

# 准备kubectl配置文件并进行角色绑定


kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes --kubeconfig=/root/.kube/config
```

### 2.7 kube-controller-manager

#### 2.7.1 配置证书文件

```
cat > kube-controller-manager-csr.json << "EOF"
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
      "127.0.0.1",
      "192.168.139.131"
    ],
    "names": [
      {
        "C": "CN",
        "ST": "Beijing",
        "L": "Beijing",
        "O": "system:kube-controller-manager",
        "OU": "system"
      }
    ]
}
EOF



cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

#### 2.7.2 创建clusterrolebinding

```
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.139.131:6443 --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager --client-certificate=kube-controller-manager.pem --client-key=kube-controller-manager-key.pem --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context system:kube-controller-manager --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig

cp kube-controller-manager*.pem /etc/kubernetes/ssl/
cp kube-controller-manager.kubeconfig /etc/kubernetes/
```

#### 2.7.3 启动kube-controller-manager

```
 cd /usr/local/kubernetes-1.26.7/cmd/kube-controller-manager
 
 go run controller-manager.go --v=2 \
      --bind-address=127.0.0.1 \
      --root-ca-file=/etc/kubernetes/ssl/ca.pem \
      --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
      --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
      --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
      --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \
      --leader-elect=true \
      --use-service-account-credentials=true \
      --node-monitor-grace-period=40s \
      --node-monitor-period=5s \
      --pod-eviction-timeout=2m0s \
      --controllers=*,bootstrapsigner,tokencleaner \
      --allocate-node-cidrs=true \
      --service-cluster-ip-range=10.96.0.0/12,fd00:1111::/112 \
      --cluster-cidr=172.16.0.0/12,fc00:2222::/112 \
      --node-cidr-mask-size-ipv4=24 \
      --node-cidr-mask-size-ipv6=120 
```

### 2.8 kube-scheduler

#### 2.8.1 证书文件准备

```
cat > kube-scheduler-csr.json << "EOF"
{
    "CN": "system:kube-scheduler",
    "hosts": [
      "127.0.0.1",
      "192.168.139.131"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "ST": "Beijing",
        "L": "Beijing",
        "O": "system:kube-scheduler",
        "OU": "system"
      }
    ]
}
EOF



cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler

kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.139.131:6443 --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler --client-certificate=kube-scheduler.pem --client-key=kube-scheduler-key.pem --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context system:kube-scheduler --cluster=kubernetes --user=system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig

cp kube-scheduler*.pem /etc/kubernetes/ssl/
cp kube-scheduler.kubeconfig /etc/kubernetes/
```

#### 2.8.2 启动kube-scheduler

```
cd /usr/local/kubernetes-1.26.7/cmd/kube-scheduler

go run scheduler.go --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig --leader-elect=true  --v=2
```

### 2.9 取消master节点的不可调度

```
# 获取当前节点的名字
kubectl get node

root@jcrose:/data/k8s-work# kubectl get node
NAME     STATUS     ROLES    AGE   VERSION
jcrose   NotReady   <none>   31m   v0.0.0-master+84e1fc493a47446df2e155e70fca768d2653a398
root@jcrose:/data/k8s-work# kubectl describe node jcrose|grep Taint
Taints:             node.kubernetes.io/not-ready:NoSchedule
root@jcrose:/data/k8s-work# kubectl taint node jcrose node.kubernetes.io/not-ready:NoSchedule-
node/jcrose untainted
```

### 2.10 检测 etcd,controller-manager,scheduler的组件状态

```
kubectl get cs

Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE   ERROR
controller-manager   Healthy   ok        
scheduler            Healthy   ok        
etcd-0               Healthy   
```

## 三 配置kubelet

### 3.1 配置runtime容器

#### 3.1.1 下载软件包containerd

```powershell
wget https://ghproxy.com/https://github.com/containerd/containerd/releases/download/v1.6.10/cri-containerd-cni-1.6.10-linux-amd64.tar.gz
wget https://ghproxy.com/https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
wget https://ghproxy.com/https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.26.0/crictl-v1.26.0-linux-amd64.tar.gz

#创建cni插件所需目录
mkdir -p /etc/cni/net.d /opt/cni/bin 

#解压cni二进制包
tar xf cni-plugins-linux-amd64-v*.tgz -C /opt/cni/bin/

#解压
tar -xzf cri-containerd-cni-*-linux-amd64.tar.gz -C /

#创建服务启动文件
cat > /etc/systemd/system/containerd.service <<EOF
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF
```

#### 3.1.2 配置Containerd所需的模块

```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
 
#加载模块
systemctl restart systemd-modules-load.service

```

#### 3.1.3 配置Containerd所需的内核

```
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# 加载内核

sysctl --system
```

#### 3.1.4 创建Containerd的配置文件

```
# 创建默认配置文件
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

# 修改Containerd的配置文件
sed -i "s#SystemdCgroup\ \=\ false#SystemdCgroup\ \=\ true#g" /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep SystemdCgroup

sed -i "s#registry.k8s.io#registry.cn-hangzhou.aliyuncs.com/chenby#g" /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep sandbox_image

sed -i "s#config_path\ \=\ \"\"#config_path\ \=\ \"/etc/containerd/certs.d\"#g" /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep certs.d

mkdir /etc/containerd/certs.d/docker.io -pv

cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://hub-mirror.c.163.com"]
  capabilities = ["pull", "resolve"]
EOF
```



#### 3.1.5 设置服务为开机自动



```powershell
systemctl daemon-reload
systemctl enable --now containerd
systemctl restart containerd
```



#### 3.1.6 配置crictl客户端连接的运行时位置

```powershell
tar xf crictl-v*-linux-amd64.tar.gz -C /usr/bin/
#生成配置文件
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

#测试
systemctl restart  containerd
crictl info
```





### 3.2 kubelet

主要用于k8s-apiserver与 节点node进行交互

#### 3.2.1 创建配置文件

创建kubelet-bootstrap.kubeconfig

```
BOOTSTRAP_TOKEN=$(awk -F "," '{print $1}' /etc/kubernetes/ssl/token.csv)

kubectl config set-cluster kubernetes     \
--certificate-authority=/etc/kubernetes/ssl/ca.pem     \
--embed-certs=true     --server=https://192.168.139.131:6443     \
--kubeconfig=kubelet-bootstrap.kubeconfig

kubectl config set-credentials tls-bootstrap-token-user     \
--token=${BOOTSTRAP_TOKEN} \
--kubeconfig=kubelet-bootstrap.kubeconfig

kubectl config set-context tls-bootstrap-token-user@kubernetes     \
--cluster=kubernetes     \
--user=tls-bootstrap-token-user     \
--kubeconfig=kubelet-bootstrap.kubeconfig

kubectl config use-context tls-bootstrap-token-user@kubernetes     \
--kubeconfig=kubelet-bootstrap.kubeconfig


cp kubelet-bootstrap.kubeconfig /etc/kubernetes
```

#### 3.2.2 创建kubelet.json文件

```
cat > kubelet.json << "EOF"
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/etc/kubernetes/ssl/ca.pem"
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
  "address": "192.168.139.131",
  "port": 10250,
  "readOnlyPort": 10255,
  "cgroupDriver": "systemd",                    
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "clusterDomain": "cluster.local.",
  "clusterDNS": ["10.96.0.2"]
}
EOF

# 同步文件到集群节点
cp kubelet-bootstrap.kubeconfig /etc/kubernetes/
cp kubelet.json /etc/kubernetes/
```

#### 3.2.3 启动kubelet

```
cd /usr/local/kubernetes-1.26.7/cmd/kubelet
go run kubelet.go  --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.kubeconfig \
    --kubeconfig=/root/.kube/config \
    --config=/etc/kubernetes/kubelet.json \
    --container-runtime-endpoint=unix:///run/containerd/containerd.sock  \
    --node-labels=node.kubernetes.io/node=
   
```

### 3.3 配置kube-proxy

#### 3.3.1 创建kube-proxy配置文件

```
cat > kube-proxy-csr.json << "EOF"
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "kubemsb",
      "OU": "CN"
    }
  ]
}
EOF
```

#### 3.3.2 生成证书

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://192.168.139.131:6443 --kubeconfig=kube-proxy.kubeconfig


kubectl config set-credentials kube-proxy  \
  --client-certificate=kube-proxy.pem     \
  --client-key=kube-proxy-key.pem     \
  --embed-certs=true     \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context kube-proxy@kubernetes    \
  --cluster=kubernetes     \
  --user=kube-proxy     \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context kube-proxy@kubernetes  --kubeconfig=kube-proxy.kubeconfig
```

#### 3.3.3 创建配置文件

```
cat > kube-proxy.yaml << EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 172.16.0.0/12,fc00:2222::/112
configSyncPeriod: 15m0s
conntrack:
  max: null
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  masqueradeAll: true
  minSyncPeriod: 5s
  scheduler: "rr"
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms

EOF
```



#### 3.3.4 移动配置文件

```powershell
cp kube-proxy.yaml kube-proxy.kubeconfig /etc/kubernetes
cp kube-proxy.pem kube-proxy-key.pem /etc/kubernetes/ssl/
```





#### 3.3.5启动kube-proxy

```
cd /usr/local/kubernetes-1.26.7/cmd/kube-proxy
go run  proxy.go --config=/etc/kubernetes/kube-proxy.yaml --v=2
```



### 3.4 安装网络插件caclico



#### 3.4.1 下载yaml文件

```
wget https://ghproxy.com/https://github.com/cby-chen/Kubernetes/blob/main/calico/calico.yaml
sed -i "s#docker.io/calico/#registry.cn-hangzhou.aliyuncs.com/chenby/#g" calico.yaml 


kubectl apply -f calico.yaml
```



### 3.5 查看服务状态

```powershell
root@jcrose:/data/k8s-work# kubectl get node
NAME     STATUS   ROLES    AGE     VERSION
jcrose   Ready    <none>   2m55s   v0.0.0-master+84e1fc493a47446df2e155e70fca768d2653a398
root@jcrose:/data/k8s-work# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE   ERROR
scheduler            Healthy   ok        
controller-manager   Healthy   ok        
etcd-0               Healthy             
root@jcrose:/data/k8s-work# kubectl cluster-info
Kubernetes control plane is running at https://192.168139.131:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.


root@jcrose:/data/k8s-work# kubectl get po -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7c6f5c46cc-27ftx   1/1     Running   0          52s
calico-node-7d66n                          1/1     Running   0          52s
calico-typha-d76b77b7f-wmtfv               1/1     Running   0          52s
```



## 四 启动命令

### 4.1 kube-apiserver

```powershell
cd /usr/local/kubernetes-1.26.7/cmd/kube-apiserver

# 这里的ip地址就是本机的ip

go run apiserver.go --anonymous-auth=true \
  --v=2 \
  --allow-privileged=true \
  --bind-address=0.0.0.0 \
  --secure-port=6443 \
  --advertise-address=192.168.139.131 \
  --service-cluster-ip-range=10.96.0.0/12,fd00:1111::/112 \
  --service-node-port-range=30000-32767 \
  --etcd-servers=https://127.0.0.1:2379 \
  --etcd-cafile=/etc/etcd/ssl/ca.pem \
  --etcd-certfile=/etc/etcd/ssl/etcd.pem \
  --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem  \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem \
  --kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \
  --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
  --authorization-mode=Node,RBAC \
  --runtime-config=api/all=true \
  --enable-bootstrap-token-auth \
  --token-auth-file=/etc/kubernetes/ssl/token.csv \
  --requestheader-allowed-names=aggregator  \
  --requestheader-group-headers=X-Remote-Group  \
  --requestheader-extra-headers-prefix=X-Remote-Extra-  \
  --requestheader-username-headers=X-Remote-User \
  --enable-aggregator-routing=true
```



### 4.2 kube-controller-manager

```powershell
 cd /usr/local/kubernetes-1.26.7/cmd/kube-controller-manager
 
 go run controller-manager.go --v=2 \
      --bind-address=127.0.0.1 \
      --root-ca-file=/etc/kubernetes/ssl/ca.pem \
      --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
      --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
      --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
      --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \
      --leader-elect=true \
      --use-service-account-credentials=true \
      --node-monitor-grace-period=40s \
      --node-monitor-period=5s \
      --pod-eviction-timeout=2m0s \
      --controllers=*,bootstrapsigner,tokencleaner \
      --allocate-node-cidrs=true \
      --service-cluster-ip-range=10.96.0.0/12,fd00:1111::/112 \
      --cluster-cidr=172.16.0.0/12,fc00:2222::/112 \
      --node-cidr-mask-size-ipv4=24 \
      --node-cidr-mask-size-ipv6=120 
```



### 4.3 kube-shceduler

```powershell
cd /usr/local/kubernetes-1.26.7/cmd/kube-scheduler

go run scheduler.go --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig --leader-elect=true  --v=2
```



### 4.4 kubelet

```powershell
cd /usr/local/kubernetes-1.26.7/cmd/kubelet
go run kubelet.go  --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.kubeconfig \
    --kubeconfig=/root/.kube/config \
    --config=/etc/kubernetes/kubelet.json \
    --container-runtime-endpoint=unix:///run/containerd/containerd.sock  \
    --node-labels=node.kubernetes.io/node=
```



### 4.5 kube-proxy

```powershell
cd /usr/local/kubernetes-1.26.7/cmd/kube-proxy
go run  proxy.go --config=/etc/kubernetes/kube-proxy.yaml --v=2
```

