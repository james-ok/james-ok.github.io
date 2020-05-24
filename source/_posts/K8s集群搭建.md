---
title: K8s集群搭建
date: 2020-05-19 20:54:07
tags:
---

服务器规划
192.168.10.41	k8s-m1
192.168.10.42	k8s-w1
192.168.10.43	k8s-w2


软件版本
Kubernetes 1.14.2
Docker 18.09 (docker使用官方的脚本安装，后期可能升级为新的版本，但是不影响)
Etcd 3.3.13
Flanneld 0.11.0


## 准备工作
1. hostname修改
所有节点都需要修改hostname
```
$ hostnamectl set-hostname k8s-m1
$ hostnamectl set-hostname k8s-w1
$ hostnamectl set-hostname k8s-w2
```
2. hosts修改
所有节点都需要映射主机名和IP地址
```
$ cat >> /etc/hosts <<EOF
192.168.10.41   k8s-m1  etcd kube-apiserver kube-scheduler kube-controller-manager kubectl cfssl(该工具可以随意在任何节点上安装)
192.168.10.42   k8s-w1  etcd kubelet kube-proxy docker flannel
192.168.10.43   k8s-w2  etcd kubelet kube-proxy docker flannel
EOF
```

3. 关闭防火墙等
```
# 关闭防火墙
$ systemctl stop firewalld && systemctl disable firewalld

# 重置iptables
$ iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT

# 关闭swap
$ swapoff -a
$ sed -i '/swap/s/^\\(.*\\)$/#\1/g' /etc/fstab

# 关闭selinux
$ setenforce 0

# 关闭dnsmasq(否则可能导致docker容器无法解析域名)
$ service dnsmasq stop && systemctl disable dnsmasq
```

4. 在`k8s-m1`节点上安装cfssl
```
$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
$ chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
$ mv cfssl_linux-amd64 /usr/local/bin/cfssl
$ mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
$ mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

5. 下载软件包
etcd[下载](https://github.com/etcd-io/etcd/releases/download/v3.3.13/etcd-v3.3.13-linux-amd64.tar.gz)
Kubernetes[下载](https://storage.googleapis.com/kubernetes-release/release/v1.14.2/kubernetes-server-linux-amd64.tar.gz)
flannel[下载](https://github.com/coreos/flannel/releases/download/v0.12.0/flannel-v0.12.0-linux-amd64.tar.gz)

## 安装ETCD
这里搭建一个Https的Etcd集群，etcd分布在三个节点中

1. 生成证书
创建证书目录，所有的证书都在该目录下生成
```
$ mkdir /root/developer/k8s/ssl/{etcd-cert,k8s-cert}
$ cd /root/developer/k8s/ssl/etcd-cert
```
创建CA配置json文件（这里也可以使用默认配置文件进行修改）
```
$ cat > ca-config.json <<EOF
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
```
创建证书请求json文件
```
$ cat > ca-csr.json <<EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "ShenZhen",
            "ST": "GuangDong"
        }
    ]
}
EOF
```
生成证书
```
$ cfssl gencert --initca ca-csr.json | cfssljson --bare ca
```
这里会生成三个文件，分别是`ca.csr`,`ca.pem`,`ca-key.pem`
创建Etcd证书请求文件
```
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "192.168.10.41",
    "192.168.10.42",
    "192.168.10.43"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "ShenZhen",
      "ST": "GuangDong"
    }
  ]
}

EOF
```
生成Etcd证书
```
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www etcd-csr.json | cfssljson -bare etcd
```
注意参数`-profile`该参数需要和ca-config.json文件中的`profiles`属性中对应，这里会生成三个文件，分别是`etcd.csr`,`etcd.pem`,`etcd-key.pem`，查看当前目录，证书文件应该有如下：
```
$ ls
-rw-r--r--. 1 root root  366 5月  23 09:43 ca-config.json
-rw-r--r--. 1 root root  960 5月  23 09:45 ca.csr
-rw-r--r--. 1 root root  213 5月  23 09:45 ca-csr.json
-rw-------. 1 root root 1675 5月  23 09:45 ca-key.pem
-rw-r--r--. 1 root root 1273 5月  23 09:45 ca.pem
-rw-r--r--. 1 root root 1017 5月  23 09:52 etcd.csr
-rw-r--r--. 1 root root  245 5月  23 09:50 etcd-csr.json
-rw-------. 1 root root 1679 5月  23 09:52 etcd-key.pem
-rw-r--r--. 1 root root 1346 5月  23 09:52 etcd.pem
```

2. 安装Etcd
Etcd安装很简单，解压放到指定目录即可
```
$ tar -zxvf tar -zxvf etcd-v3.3.13-linux-amd64.tar.gz
$ cp /root/downloads/etcd-v3.3.13-linux-amd64/{etcd,etcdctl} /root/developer/k8s/bins/etcd/bin/
```

3. 设置系统服务
```
$ cat > /usr/lib/systemd/system/etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/root/developer/data/etcd/
ExecStart=/root/developer/k8s/bins/etcd/bin/etcd \
  --data-dir=/root/developer/data/etcd \
  --name=etcd1 \
  --cert-file=/root/developer/k8s/bins/etcd/ssl/etcd.pem \
  --key-file=/root/developer/k8s/bins/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/root/developer/k8s/bins/etcd/ssl/ca.pem \
  --peer-cert-file=/root/developer/k8s/bins/etcd/ssl/etcd.pem \
  --peer-key-file=/root/developer/k8s/bins/etcd/ssl/etcd-key.pem \
  --peer-trusted-ca-file=/root/developer/k8s/bins/etcd/ssl/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --listen-peer-urls=https://192.168.10.41:2380 \
  --initial-advertise-peer-urls=https://192.168.10.41:2380 \
  --listen-client-urls=https://192.168.10.41:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://192.168.10.41:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster=etcd1=https://192.168.10.41:2380,etcd2=https://192.168.10.42:2380,etcd3=https://192.168.10.43:2380 \
  --initial-cluster-state=new
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
这里需要注意的点是：以上所有的操作三个节点都是一致的，唯独在创建系统服务时，这里的`--name`/`--listen-peer-urls`/`--initial-advertise-peer-urls`/`--listen-client-urls`/`--advertise-client-urls`
参数需要根据当前系统的角色和ip来决定

4. 其他两个节点部署
将生成的证书`ca.pem`/`etcd.pem`/`etcd-key.pem`分别copy到剩余两个节点
将etcd的二进制文件copy到剩余两个节点
将`etcd.service`文件copy到剩余两个节点，并根据情况修改


4. 验证Etcd集群
分别启动三个节点的etcd
```
$ systemctl start etcd
```
使用客户端命令行工具查看集群状态
```
$ /root/developer/k8s/bins/etcd/bin/etcdctl \
  --ca-file=/root/developer/k8s/bins/etcd/ssl/ca.pem \
  --cert-file=/root/developer/k8s/bins/etcd/ssl/etcd.pem \
  --key-file=/root/developer/k8s/bins/etcd/ssl/etcd-key.pem \
  --endpoints=https://192.168.10.41:2379,https://192.168.10.42:2379,https://192.168.10.43:2379 \
  cluster-health
member 6d4f7e638629524f is healthy: got healthy result from https://192.168.10.43:2379
member 90f48187357a01e4 is healthy: got healthy result from https://192.168.10.42:2379
member b9d45e1f7960baa1 is healthy: got healthy result from https://192.168.10.41:2379
cluster is healthy
```
至此，Etcd集群部署完成

## 安装docker
Docker只需要在node节点上安装，不需要在master节点上安装
```
$ yum install -y yum-utils device-mapper-persistent-data lvm2
$ yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
$ yum install -y docker-ce-18.09.0-3.el7 docker-ce-cli-18.09.0-3.el7 containerd.io
```
配置镜像加速器
```
$ sudo mkdir -p /etc/docker
$ sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://gqjyyepn.mirror.aliyuncs.com"]
}
EOF
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

## 部署Flannel
Flannel的作用是跨主机节点间docker容器的通信
1. 首先将分配给flannel的子网段写入到etcd中，避免多个flannel节点间ip冲突
```
$ /root/developer/k8s/bins/etcd/bin/etcdctl \
    --ca-file=/root/developer/k8s/bins/etcd/ssl/ca.pem \
    --cert-file=/root/developer/k8s/bins/etcd/ssl/etcd.pem \
    --key-file=/root/developer/k8s/bins/etcd/ssl/etcd-key.pem \
    --endpoints=https://192.168.10.41:2379,https://192.168.10.42:2379,https://192.168.10.43:2379 \
    set /coreos.com/network/config  '{ "Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'
```

2. 部署flannel
1). 将解压的flannel二进制文件（包含`flanneld`/`mk-docker-opts.sh`）放到`/root/developer/k8s/bins/flannel/bin`目录下
2). 创建配置文件
```
$ mkdir /root/developer/k8s/bins/flannel/cfg
$ cat > /root/developer/k8s/bins/flannel/cfg/flannel  <<EOF
FLANNEL_ETCD="--etcd-endpoints=https://192.168.10.41:2379,https://192.168.10.42:2379,https://192.168.10.43:2379"
FLANNEL_ETCD_CAFILE="--etcd-cafile=/root/developer/k8s/bins/etcd/ssl/ca.pem"
FLANNEL_ETCD_CERTFILE="--etcd-certfile=/root/developer/k8s/bins/etcd/ssl/etcd.pem"
FLANNEL_ETCD_KEYFILE="--etcd-keyfile=/root/developer/k8s/bins/etcd/ssl/etcd-key.pem"

EOF
```

3. 创建系统服务
```
$ cat > /usr/lib/systemd/system/flannel.service  <<EOF
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
Before=docker.service

[Service]
EnvironmentFile=-/root/developer/k8s/bins/flannel/cfg/flannel
ExecStart=/root/developer/k8s/bins/flannel/bin/flanneld ${FLANNEL_ETCD} ${FLANNEL_ETCD_CAFILE} ${FLANNEL_ETCD_CERTFILE} ${FLANNEL_ETCD_KEYFILE}
ExecStartPost=/root/developer/k8s/bins/flannel/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker

Type=notify

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF
```
mk-docker-opts.sh脚本将分配给flannel的Pod子网网段信息写入到/run/flannel/docker文件中，后续docker启动时使用这个文件中参数值设置docker0网桥；


3. 配置docker使用flannel分配的子网
修改docker.service文件
```
$ vim /usr/lib/systemd/system/docker.service
```
1). 在Unit段中的After后面添加flannel.service参数，在Wants下面添加`Requires=flannel.service`.
2). \[Service]段中Type后面添加`EnvironmentFile=-/run/flannel/docker`段，在ExecStart后面添加$DOCKER_OPTS参数.
如下：
```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
BindsTo=containerd.service
After=network-online.target firewalld.service
Wants=network-online.target
Requires=flannel.service

[Service]
Type=notify
EnvironmentFile=-/run/flannel/docker
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS -H unix://
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

```
重载配置文件
```
$ systemctl daemon-reload
```

4. 启动flannel,重启docker
```
$ systemctl start flannel
$ systemctl restart docker
```

4. 启动验证
```
ip a
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:6d:5b:7d:f3 brd ff:ff:ff:ff:ff:ff
    inet 172.17.51.1/24 brd 172.17.51.255 scope global docker0
       valid_lft forever preferred_lft forever
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
    link/ether 76:8b:ca:35:c2:34 brd ff:ff:ff:ff:ff:ff
    inet 172.17.51.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::748b:caff:fe35:c234/64 scope link 
       valid_lft forever preferred_lft forever
```
如果发现docker0和flannel出于同一网段则证明flannel和docker搭建成功
我们可以在etcd中查看到当前docker的子网ip属于哪一台物理节点
```
$ /root/developer/k8s/bins/etcd/bin/etcdctl \
--ca-file=/root/developer/k8s/bins/etcd/ssl/ca.pem \
--cert-file=/root/developer/k8s/bins/etcd/ssl/etcd.pem \
--key-file=/root/developer/k8s/bins/etcd/ssl/etcd-key.pem \
--endpoints=https://192.168.10.41:2379,https://192.168.10.42:2379,https://192.168.10.43:2379 \
get /coreos.com/network/subnets/172.17.51.0-24
{"PublicIP":"192.168.10.42","BackendType":"vxlan","BackendData":{"VtepMAC":"76:8b:ca:35:c2:34"}}
```

## 部署kube-api-server
1. 生成证书
创建kube-api-server-csr.json文件(这里将etcd中的ca-config.json/ca.pem和ca-key.pem拷贝过来，使用同一个ca就够了)
```
$ cat > /root/developer/k8s/ssl/k8s-cert/kube-api-server-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "192.168.10.41",
    "192.168.10.51",
    "192.168.10.52",
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
      "ST": "GuangDong",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
生成证书
```
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www kube-api-server-csr.json | cfssljson -bare server
```

2. 将生成的ca.pem/ca-key.pem/server.pem以及server-key.pem文件copy到相应的目录
```
$ mkdir ../../bins/kube-api-server/{cfg,bin,ssl} -p
$ cp {ca,ca-key,server,server-key}.pem ../../bins/kube-api-server/ssl/
```

3. 生成Token文件(该token文件用于kubelet)
```
$ head -c 16 /dev/urandom | od -An -t x | tr -d ' '
dc7903bc34d75a708526369ceeb8250e
$ cat > /root/developer/k8s/bins/kube-api-server/cfg/token.csv <<EOF
dc7903bc34d75a708526369ceeb8250e,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```
token.csv文件第一个参数标识一个tokenId，第二个参数标识一个用户，第三个标识用户组，第四个标识当前加入到k8s的角色
将该配置文件放到cfg目录下面
```
$ mv token.csv /root/developer/k8s/bins/kube-api-server/cfg/
```

4. 创建秘钥key
```
$ head -c 32 /dev/urandom | base64
ulezRZjwbnjnODMzEKoHPiTOL/d+Z2TuGCgZxui4zRQ=
```
5. 创建加密配置文件
```

$ cat > /root/developer/k8s/bins/kube-api-server/cfg/encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ulezRZjwbnjnODMzEKoHPiTOL/d+Z2TuGCgZxui4zRQ=
      - identity: {}
EOF
```
6. 创建日志目录，将k8s的日志输出到`/var/log/k8s`下
```
$ mkdir /var/log/k8s
```
7. 创建系统服务
```
$ cat > /usr/lib/systemd/system/kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
 
[Service]
ExecStart=/root/developer/k8s/bins/kube-api-server/bin/kube-apiserver \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --anonymous-auth=false \\
  --experimental-encryption-provider-config=/root/developer/k8s/bins/kube-api-server/cfg/encryption-config.yaml \\
  --advertise-address=192.168.10.41 \\
  --bind-address=192.168.10.41 \\
  --insecure-port=0 \\
  --authorization-mode=Node,RBAC \\
  --runtime-config=api/all \\
  --enable-bootstrap-token-auth \\
  --token-auth-file=/root/developer/k8s/bins/kube-api-server/cfg/token.csv \\
  --service-cluster-ip-range=10.254.0.0/16 \\
  --tls-cert-file=/root/developer/k8s/bins/kube-api-server/ssl/server.pem \\
  --tls-private-key-file=/root/developer/k8s/bins/kube-api-server/ssl/server-key.pem \\
  --client-ca-file=/root/developer/k8s/bins/kube-api-server/ssl/ca.pem \\
  --kubelet-client-certificate=/root/developer/k8s/bins/kube-api-server/ssl/server.pem \\
  --kubelet-client-key=/root/developer/k8s/bins/kube-api-server/ssl/server-key.pem \\
  --service-account-key-file=/root/developer/k8s/bins/kube-api-server/ssl/ca-key.pem \\
  --etcd-cafile=/root/developer/k8s/bins/etcd/ssl/ca.pem \\
  --etcd-certfile=/root/developer/k8s/bins/etcd/ssl/etcd.pem \\
  --etcd-keyfile=/root/developer/k8s/bins/etcd/ssl/etcd-key.pem \\
  --etcd-servers=https://192.168.10.41:2379,https://192.168.10.42:2379,https://192.168.10.43:2379 \\
  --enable-swagger-ui=true \\
  --allow-privileged=true \\
  --apiserver-count=2 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/k8s/kube-apiserver-audit.log \\
  --event-ttl=1h \\
  --alsologtostderr=true \\
  --logtostderr=false \\
  --log-dir=/var/log/k8s \\
  --v=2
Restart=on-failure
RestartSec=5
Type=notify
#User=k8s
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
EOF
```

## 部署kubectl
1. 将下载好的kubectl二进制文件直接放到`/usr/bin`目录下
```
$ cp /root/downloads/kubernetes/server/bin/kubectl /usr/bin/
```
2. 生成证书
生成证书请求文件
```
$ cat > /root/developer/k8s/ssl/k8s-cert/admin-csr.json <<EOF
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
      "ST": "GuangDong",
      "L": "ShenZhen",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
```
生成证书
```
$ cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=www admin-csr.json | cfssljson -bare admin
```
3. 创建安装目录，并且将ssl复制到安装目录
```
$ mkdir /root/developer/k8s/bins/kubectl/{ssl,cfg} -p
$ cp {ca,ca-key,admin,admin-key}.pem /root/developer/k8s/bins/kubectl/ssl/
```

4. 生成配置文件
```
$ cd /root/.kube/
$ kubectl config set-cluster kubernetes \
    --certificate-authority=/root/developer/k8s/bins/kubectl/ssl/ca.pem \
    --embed-certs=true \
    --server=https://192.168.10.41:6443 \
    --kubeconfig=kube.config
$ kubectl config set-credentials admin \
    --client-certificate=/root/developer/k8s/bins/kubectl/ssl/admin.pem \
    --client-key=/root/developer/k8s/bins/kubectl/ssl/admin-key.pem \
    --embed-certs=true \
    --kubeconfig=kube.config
$ kubectl config set-context kubernetes \
    --cluster=kubernetes \
    --user=admin \
    --kubeconfig=kube.config
$ kubectl config use-context kubernetes --kubeconfig=kube.config
$ mv kube.config config
$ cat ~/.kube/config
```

## 部署kube-controller-manager
1. 创建证书请求json文件
```
$ cat > /root/developer/k8s/ssl/k8s-cert/kube-controller-manager-csr.json <<EOF
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
        "127.0.0.1",
        "192.168.10.41",
        "node01.k8s.com",
        "node02.k8s.com",
        "node03.k8s.com"
    ],
    "names": [
      {
        "C": "CN",
        "ST": "GuangDong",
        "L": "ShenZhen",
        "O": "system:kube-controller-manager",
        "OU": "System"
      }
    ]
}
```
CN和O均为system:kube-controller-manager，kubernetes内置的ClusterRoleBindings system:kube-controller-manager赋予kube-controller-manager工作所需权限
生成证书和私钥
```
$ cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=www kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```
2. 将二进制文件和证书文件copy到相应安装目录
```
$ mkdir /root/developer/k8s/bins/kube-controller-manager/{cfg,bin,ssl} -p
$ cp {ca,ca-key,kube-controller-manager,kube-controller-manager-key}.pem /root/developer/k8s/bins/kube-controller-manager/ssl/
$ cp /root/downloads/kubernetes/server/bin/kube-controller-manager /root/developer/k8s/bins/kube-controller-manager/bin/
```
3. 创建kubeconfig文件
kube-controller-manager使用kubeconfig文件访问apiserver，该文件提供了apiserver地址、嵌入的CA证书和kube-controller-manager证书
```
$ cd /root/developer/k8s/bins/kube-controller-manager/cfg/
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/root/developer/k8s/bins/kube-controller-manager/ssl/ca.pem \
  --embed-certs=true \
  --server=https://192.168.10.41:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig
$ kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=/root/developer/k8s/bins/kube-controller-manager/ssl/kube-controller-manager.pem \
  --client-key=/root/developer/k8s/bins/kube-controller-manager/ssl/kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig
$ kubectl config set-context system:kube-controller-manager \
  --cluster=kubernetes \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig
$ kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
```
4. 创建系统服务
```
$ cat > /usr/lib/systemd/system/kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
[Service]
ExecStart=/root/developer/k8s/bins/kube-controller-manager/bin/kube-controller-manager \\
  --profiling \\
  --cluster-name=kubernetes \\
  --controllers=*,bootstrapsigner,tokencleaner \\
  --kube-api-qps=1000 \\
  --kube-api-burst=2000 \\
  --leader-elect \\
  --use-service-account-credentials\\
  --concurrent-service-syncs=2 \\
  --bind-address=0.0.0.0 \\
  --secure-port=10252 \\
  --tls-cert-file=/root/developer/k8s/bins/kube-controller-manager/ssl/kube-controller-manager.pem \\
  --tls-private-key-file=/root/developer/k8s/bins/kube-controller-manager/ssl/kube-controller-manager-key.pem \\
  --port=0 \\
  --authentication-kubeconfig=/root/developer/k8s/bins/kube-controller-manager/cfg/kube-controller-manager.kubeconfig \\
  --client-ca-file=/root/developer/k8s/bins/kube-controller-manager/ssl/ca.pem \\
  --requestheader-allowed-names="" \\
  --requestheader-client-ca-file=/root/developer/k8s/bins/kube-controller-manager/ssl/ca.pem \\
  --requestheader-extra-headers-prefix="X-Remote-Extra-" \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-username-headers=X-Remote-User \\
  --authorization-kubeconfig=/root/developer/k8s/bins/kube-controller-manager/cfg/kube-controller-manager.kubeconfig \\
  --cluster-signing-cert-file=/root/developer/k8s/bins/kube-controller-manager/ssl/ca.pem \\
  --cluster-signing-key-file=/root/developer/k8s/bins/kube-controller-manager/ssl/ca-key.pem \\
  --experimental-cluster-signing-duration=876000h \\
  --horizontal-pod-autoscaler-sync-period=10s \\
  --concurrent-deployment-syncs=10 \\
  --concurrent-gc-syncs=30 \\
  --node-cidr-mask-size=24 \\
  --service-cluster-ip-range=10.254.0.0/16 \\
  --pod-eviction-timeout=6m \\
  --terminated-pod-gc-threshold=10000 \\
  --root-ca-file=/root/developer/k8s/bins/kube-controller-manager/ssl/ca.pem \\
  --service-account-private-key-file=/root/developer/k8s/bins/kube-controller-manager/ssl/ca-key.pem \\
  --kubeconfig=/root/developer/k8s/bins/kube-controller-manager/cfg/kube-controller-manager.kubeconfig \\
  --logtostderr=false \\
  --log-dir=/var/log/k8s \\
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
```
service-cluster-ip-range ：指定 Service Cluster IP 网段，必须和 kube-apiserver 中的同名参数一致；

## 部署kube-scheduler
1. 生成证书
创建证书请求文件
```
$ cat > /root/developer/k8s/ssl/k8s-cert/kube-scheduler-csr.json <<EOF
{
    "CN": "system:kube-scheduler",
    "hosts": [
      "127.0.0.1",
      "192.168.10.41"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "ST": "GuangDong",
        "L": "ShenZhen",
        "O": "system:kube-scheduler",
        "OU": "System"
      }
    ]
}
EOF
```
生成证书
```
$ cfssl gencert -ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=www kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```
2. 将下载的二进制文件和ssl文件copy到安装目录
```
$ mkdir /root/developer/k8s/bins/kube-scheduler/{cfg,bin,ssl} -p
$ cp /root/downloads/kubernetes/server/bin/kube-scheduler /root/developer/k8s/bins/kube-scheduler/bin/
$ cp {ca,ca-key,kube-scheduler,kube-scheduler-key}.pem /root/developer/k8s/bins/kube-scheduler/ssl/
```
3. 生成配置文件
```
$ cd /root/developer/k8s/bins/kube-scheduler/cfg
# 创建kubeconfig
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/root/developer/k8s/bins/kube-scheduler/ssl/ca.pem \
  --embed-certs=true \
  --server=https://192.168.10.41:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-credentials system:kube-scheduler \
  --client-certificate=/root/developer/k8s/bins/kube-scheduler/ssl/kube-scheduler.pem \
  --client-key=/root/developer/k8s/bins/kube-scheduler/ssl/kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-context system:kube-scheduler \
  --cluster=kubernetes \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig 
```
4. 创建系统服务
```
$ cat > /usr/lib/systemd/system/kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/root/developer/k8s/bins/kube-scheduler/bin/kube-scheduler \\
  --address=127.0.0.1 \\
  --kubeconfig=/root/developer/k8s/bins/kube-scheduler/cfg/kube-scheduler.kubeconfig \\
  --leader-elect=true \\
  --alsologtostderr=true \\
  --logtostderr=false \\
  --log-dir=/var/log/k8s \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
5. 启动
```
$ systemctl daemon-reload
$ systemctl enable kube-scheduler
$ systemctl start kube-scheduler
$ systemctl status kube-scheduler
```

## 部署kubelet（Worker节点）
1. 为kubelet创建一个bootstrap.kubeconfig文件，这里先在master节点上执行，然后拷贝到worker节点，因为只有master节点上安装了kubectl
```
# 设置集群参数
$ kubectl config set-cluster kubernetes \
      --certificate-authority=ca.pem \
      --embed-certs=true \
      --server=https://192.168.10.41:6443 \
      --kubeconfig=kubelet-bootstrap.kubeconfig

# 设置客户端认证参数
$ kubectl config set-credentials kubelet-bootstrap \
      --token=dc7903bc34d75a708526369ceeb8250e \
      --kubeconfig=kubelet-bootstrap.kubeconfig

# 设置上下文参数
$ kubectl config set-context default \
      --cluster=kubernetes \
      --user=kubelet-bootstrap \
      --kubeconfig=kubelet-bootstrap.kubeconfig

# 设置默认上下文
$ kubectl config use-context default --kubeconfig=kubelet-bootstrap.kubeconfig

$ scp kubelet-bootstrap.kubeconfig root@192.168.10.42:/root/developer/k8s/bins/kubelet/cfg
$ scp kubelet-bootstrap.kubeconfig root@192.168.10.43:/root/developer/k8s/bins/kubelet/cfg
```
创建kubelet.config.json文件
```
$ cat > /root/developer/k8s/bins/kubelet/cfg/kubelet.config <<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 192.168.10.43
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS: ["10.254.0.2"]
clusterDomain: cluster.local.
failSwapOn: false
authentication:
  anonymous:
    enabled: true
EOF
```
2. 将下载的二进制文件以及证书文件copy到安装目录
```
$ cp /root/downloads/kubernetes/server/bin/kubelet /root/developer/k8s/bins/kubelet/bin
#这里从etcd里面copy过来，因为这里的ca只有一个
$ cp /root/developer/k8s/bins/etcd/ssl/ca.pem /root/developer/k8s/bins/kubelet/ssl
```
3. 创建系统服务
```
$ cat > /usr/lib/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/root/developer/k8s/bins/kubelet/bin/kubelet \
  --bootstrap-kubeconfig=/root/developer/k8s/bins/kubelet/cfg/kubelet-bootstrap.kubeconfig \
  --cert-dir=/root/developer/k8s/bins/kubelet/ssl \
  --kubeconfig=/root/developer/k8s/bins/kubelet/cfg/kubelet.kubeconfig \
  --config=/root/developer/k8s/bins/kubelet/cfg/kubelet.config \
  --pod-infra-container-image=k8s.gcr.io/pause-amd64:3.1 \
  --allow-privileged=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/k8s \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
4. 由于pause镜像默认拉取需要科学上网，所有这里通过国内下载打标签的方式预先下载好镜像
```
cat > download-images.sh <<EOF
#!/bin/bash

docker pull registry.cn-hangzhou.aliyuncs.com/liuyi01/calico-node:v3.1.3
docker tag registry.cn-hangzhou.aliyuncs.com/liuyi01/calico-node:v3.1.3 quay.io/calico/node:v3.1.3
docker rmi registry.cn-hangzhou.aliyuncs.com/liuyi01/calico-node:v3.1.3

docker pull registry.cn-hangzhou.aliyuncs.com/liuyi01/calico-cni:v3.1.3
docker tag registry.cn-hangzhou.aliyuncs.com/liuyi01/calico-cni:v3.1.3 quay.io/calico/cni:v3.1.3
docker rmi registry.cn-hangzhou.aliyuncs.com/liuyi01/calico-cni:v3.1.3

docker pull registry.cn-hangzhou.aliyuncs.com/liuyi01/pause-amd64:3.1
docker tag registry.cn-hangzhou.aliyuncs.com/liuyi01/pause-amd64:3.1 k8s.gcr.io/pause-amd64:3.1
docker rmi registry.cn-hangzhou.aliyuncs.com/liuyi01/pause-amd64:3.1

docker pull registry.cn-hangzhou.aliyuncs.com/liuyi01/calico-typha:v0.7.4
docker tag registry.cn-hangzhou.aliyuncs.com/liuyi01/calico-typha:v0.7.4 quay.io/calico/typha:v0.7.4
docker rmi registry.cn-hangzhou.aliyuncs.com/liuyi01/calico-typha:v0.7.4

docker pull registry.cn-hangzhou.aliyuncs.com/liuyi01/coredns:1.1.3
docker tag registry.cn-hangzhou.aliyuncs.com/liuyi01/coredns:1.1.3 k8s.gcr.io/coredns:1.1.3
docker rmi registry.cn-hangzhou.aliyuncs.com/liuyi01/coredns:1.1.3

docker pull registry.cn-hangzhou.aliyuncs.com/liuyi01/kubernetes-dashboard-amd64:v1.8.3
docker tag registry.cn-hangzhou.aliyuncs.com/liuyi01/kubernetes-dashboard-amd64:v1.8.3 k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3
docker rmi registry.cn-hangzhou.aliyuncs.com/liuyi01/kubernetes-dashboard-amd64:v1.8.3
EOF
```
6. 将kubelet-bootstrap用户绑定到系统集群角色
```
$ kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```
7. 启动并审批csr
```
# 在master上Approve bootstrap请求
$ kubectl get csr
NAME                                                   AGE   REQUESTOR           CONDITION
node-csr-C4O9_KIek83fXKlhPjsW37KxpzBGl6CSspvsDEiBsPc   18s   kubelet-bootstrap   Pending
node-csr-Ow3aKEezOFC3bGIerrIu_olmsKEb02GNECffcfOYYZY   18s   kubelet-bootstrap   Pending
$ kubectl certificate approve node-csr-C4O9_KIek83fXKlhPjsW37KxpzBGl6CSspvsDEiBsPc
```

## 部署kube-proxy
1. 生成证书
创建证书请求文件
```
$ cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangDong",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
生成证书
```
$ cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=www  kube-proxy-csr.json | cfssljson -bare kube-proxy
```
2. 创建kubeconfig文件
```
# 创建kube-proxy.kubeconfig
$ kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.10.41:6443 \
  --kubeconfig=kube-proxy.kubeconfig
$ kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
$ kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

```
3. 将配置文件及ssl以及二进制执行文件copy到安装目录
```
$ scp kube-proxy.kubeconfig root@192.168.10.42:/root/developer/k8s/bins/kube-proxy/cfg
$ scp kube-proxy.pem kube-proxy-key.pem ca.pem root@192.168.10.42:/root/developer/k8s/bins/kube-proxy/ssl
$ scp kube-proxy.kubeconfig root@192.168.10.43:/root/developer/k8s/bins/kube-proxy/cfg
$ scp kube-proxy.pem kube-proxy-key.pem ca.pem root@192.168.10.43:/root/developer/k8s/bins/kube-proxy/ssl
$ cp /root/downloads/kubernetes/server/bin/kube-proxy /root/developer/k8s/bins/kube-proxy/bin
```
4. 创建配置文件
```
$ cat > /root/developer/k8s/bins/kube-proxy/cfg/kube-proxy-config.yaml <<EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 192.168.10.43
clientConnection:
  kubeconfig: /root/developer/k8s/bins/kube-proxy/cfg/kube-proxy.kubeconfig
clusterCIDR: 10.254.0.0/16
healthzBindAddress: 192.168.10.43:10256
kind: KubeProxyConfiguration
metricsBindAddress: 192.168.10.43:10249
mode: "iptables"
EOF
```
5. 创建系统服务
```
$ cat > /usr/lib/systemd/system/kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/root/developer/k8s/bins/kube-proxy/bin/kube-proxy \
  --config=/root/developer/k8s/bins/kube-proxy/cfg/kube-proxy-config.yaml \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/k8s \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
6. 启动服务
```
$ systemctl daemon-reload && systemctl enable kube-proxy && systemctl restart kube-proxy
```