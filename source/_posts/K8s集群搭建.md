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