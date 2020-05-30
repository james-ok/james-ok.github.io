---
title: K8s集群搭建
date: 2020-05-19 20:54:07
tags:
---
# Kubernetes安装部署（二进制）

## 说明

### 服务器规划

这里采用一个三节点的etcd集群，一个Master节点和两个Node节点，如下：

```
内网IP/外网IP
192.168.0.3/114.67.166.108	jd(etcd,kube-apiserver,kube-controller-manager,kube-scheduler,cfssl)
192.168.0.210/139.9.181.246	hw(etcd,kubelet,kube-proxy,flannel,docker)
172.16.0.5/106.55.0.70		tx(etcd,kubelet,kube-proxy,flannel,docker)
```

### 软件版本

```
Kubernetes 1.14.2
Docker 18.09
Etcd 3.3.13
Flanneld 0.11.0
```

### 证书

整个kubernetes（包括etcd集群）使用同一个CA，避免证书太多容易踩坑，事实上kubernetes是可以一个服务一个CA的，但官方也推荐使用同一个CA



## 准备工作

### hostname修改

```shell
hostnamectl set-hostname jd
hostnamectl set-hostname hw
hostnamectl set-hostname tx
```



### hosts文件修改

```
cat >> /etc/hosts <<EOF
114.67.166.108	jd
139.9.181.246	hw
106.55.0.70		tx
EOF
```



### 防火墙配置（关闭）

防火墙这里为了简便使用关闭的方式，也可以自己配置开放端口，如果使用云服务器，记得关闭安全组

```shell
systemctl stop firewalld && systemctl disable firewalld

iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT

swapoff -a

sed -i '/swap/s/^\\(.*\\)$/#\1/g' /etc/fstab

setenforce 0

vim /etc/selinux/config
SELINUX=disabled

service dnsmasq stop && systemctl disable dnsmasq
```



### cfssl工具安装

cfssl用于生成kubernetes所用到的证书

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64

chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64

mv cfssl_linux-amd64 /usr/local/bin/cfssl

mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

### 创建安装目录及日志目录

我这里将安装目录放到`/opt/kubernetes`下：

```shell
mkdir /opt/kubernetes/{bin,cfg,ssl,data}

mkdir /var/log/kubernetes
```

bin：用于存放二进制文件

cfg：用于存放配置文件

ssl：用于存放证书

data：用于存放数据，比如etcd的数据

/var/log/kubernetes：所有日志均放到该目录下



### 下载软件包

[etcd](https://github.com/etcd-io/etcd/releases/download/v3.3.13/etcd-v3.3.13-linux-amd64.tar.gz)

[kubernetes](https://storage.googleapis.com/kubernetes-release/release/v1.14.2/kubernetes-server-linux-amd64.tar.gz)

[flannel](https://github.com/coreos/flannel/releases/download/v0.12.0/flannel-v0.12.0-linux-amd64.tar.gz)

将下载的软件包解压后放到安装目录，我这里为`/opt/kubernetes/bin`，如下：

Master节点

```shell
[root@jd bin]# pwd
/opt/kubernetes/bin
[root@jd bin]# ll
total 331444
-rwxr-xr-x 1 root root  16927136 May 25 20:17 etcd
-rwxr-xr-x 1 root root 167595360 May 25 20:15 kube-apiserver
-rwxr-xr-x 1 root root 115612192 May 25 20:15 kube-controller-manager
-rwxr-xr-x 1 root root  39258304 May 25 20:15 kube-scheduler

```

Node节点

```shell
[root@hw bin]# pwd
/opt/kubernetes/bin
[root@hw bin]# ll
total 211776
-rwxr-xr-x 1 root root  16927136 May 25 20:17 etcd
-rwxr-xr-x 1 root root  35253112 May 25 20:19 flanneld
-rwxr-xr-x 1 root root 127981504 May 25 20:18 kubelet
-rwxr-xr-x 1 root root  36685440 May 25 20:18 kube-proxy
-rwxr-xr-x 1 root root      2139 May 25 20:19 mk-docker-opts.sh

```

Client

将`etcdctl`和`kubectl`分别放到`/usr/bin`目录下，以便在任何位置都可以访问，可以将这两个客户端放到三个节点，也可以只放到Master节点，我这里只在Master节点操作，所以Node节点上是没有的

```shell
[root@hw bin]# pwd
/usr/bin
[root@jd bin]# ll
total 132856
# 省略其他文件
-rwxr-xr-x    1 root root   13498880 May 25 20:17 etcdctl
-rwxr-xr-x    1 root root   43115328 May 26 19:38 kubectl

```



## 生成CA根证书

1. 创建CA配置文件`ca-config.json`

   ```shell
   cat > /opt/kubernetes/ssl/ca-config.json <<EOF
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
   ```

   

2. 创建CA证书请求文件`ca-csr.json`

   ```shell
   cat > /opt/kubernetes/ssl/ca-csr.json <<EOF
   {
       "CN": "CA",
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

3. 生产CA证书

   ```shell
   cfssl gencert --initca ca-csr.json | cfssljson --bare ca
   ```

4. 查看，这时会生成如下文件

   ```shell
   [root@jd ssl]# ll
   total 100
   -rw-r--r-- 1 root root  378 May 25 20:23 ca-config.json
   -rw-r--r-- 1 root root  952 May 25 20:24 ca.csr
   -rw-r--r-- 1 root root  207 May 25 20:24 ca-csr.json
   -rw------- 1 root root 1679 May 25 20:24 ca-key.pem
   -rw-r--r-- 1 root root 1261 May 25 20:24 ca.pem
   
   ```

   

## etcd部署

1. 创建etcd证书请求文件

   ```shell
   cat > /opt/kubernetes/ssl/etcd-csr.json <<EOF
   {
     "CN": "etcd",
     "hosts": [
       "114.67.166.108",
       "139.9.181.246",
       "106.55.0.70"
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

   值得注意的是这里的`hosts`需要指定三台服务器的ip

2. 生成证书

   ```shell
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
   ```

   注意参数`-profile`必须和`ca-config.json`文件中的`profiles`属性中对应

3. 查看，这时会生成如下文件：

   ```shell
   [root@jd ssl]# ll
   total 100
   -rw-r--r-- 1 root root  378 May 25 20:23 ca-config.json
   -rw-r--r-- 1 root root  952 May 25 20:24 ca.csr
   -rw-r--r-- 1 root root  207 May 25 20:24 ca-csr.json
   -rw------- 1 root root 1679 May 25 20:24 ca-key.pem
   -rw-r--r-- 1 root root 1261 May 25 20:24 ca.pem
   -rw-r--r-- 1 root root 1017 May 25 20:26 etcd.csr
   -rw-r--r-- 1 root root  244 May 25 20:26 etcd-csr.json
   -rw------- 1 root root 1679 May 25 20:26 etcd-key.pem
   -rw-r--r-- 1 root root 1338 May 25 20:26 etcd.pem
   
   ```

4. 将证书分发到各个节点

   ```shell
   scp ./* root@hw:/opt/kubernetes/ssl
   
   scp ./* root@tx:/opt/kubernetes/ssl
   ```

   确保三个节点中的证书一致（都有如上第3步中的文件）

5. 创建配置文件

   ```shell
   cat > /opt/kubernetes/cfg/etcd.conf <<EOF
   ETCD_ARGS="--data-dir=/opt/kubernetes/data/etcd \\
     --name=etcd1 \\
     --cert-file=/opt/kubernetes/ssl/etcd.pem \\
     --key-file=/opt/kubernetes/ssl/etcd-key.pem \\
     --trusted-ca-file=/opt/kubernetes/ssl/ca.pem \\
     --peer-cert-file=/opt/kubernetes/ssl/etcd.pem \\
     --peer-key-file=/opt/kubernetes/ssl/etcd-key.pem \\
     --peer-trusted-ca-file=/opt/kubernetes/ssl/ca.pem \\
     --peer-client-cert-auth \\
     --client-cert-auth \\
     --listen-peer-urls=https://192.168.0.3:2380 \\
     --listen-client-urls=https://192.168.0.3:2379,http://127.0.0.1:2379 \\
     --initial-advertise-peer-urls=https://114.67.166.108:2380 \\
     --advertise-client-urls=https://114.67.166.108:2379 \\
     --initial-cluster-token=etcd-cluster-0 \\
     --initial-cluster=etcd1=https://114.67.166.108:2380,etcd2=https://139.9.181.246:2380,etcd3=https://106.55.0.70:2380 \\
     --initial-cluster-state=new"
   EOF
   ```

   这里有几个参数需要注意一下：

   `--name`：该参数不能重复，各个节点是不一样的，且和`--initial-cluster`中的名词和地址对应

   `--listen-peer-urls`：这里填写的是当前节点内网IP

   `--listen-client-urls`：同样的，这里填写的是当前节点内网IP

   `--initial-advertise-peer-urls`：当前节点公网IP

   `--advertise-client-urls`：当前节点公网IP

   `--initial-cluster`：集群地址列表

6. 创建系统服务

   ```shell
   cat > /usr/lib/systemd/system/etcd.service <<EOF
   [Unit]
   Description=Etcd Server
   After=network.target
   After=network-online.target
   Wants=network-online.target
   Documentation=https://github.com/coreos
   
   [Service]
   Type=notify
   EnvironmentFile=/opt/kubernetes/cfg/etcd.conf
   ExecStart=/opt/kubernetes/bin/etcd $ETCD_ARGS
   Restart=on-failure
   RestartSec=5
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

   `EnvironmentFile`：需要应用的配置文件路径，`$ETCD_ARGS`这里表示使用配置文件中的配置属性

7. 启动（分别启动三个节点）

   ```shell
   systemctl daemon-reload
   
   systemctl start etcd.service
   
   systemctl enable etcd.service
   ```

8. 查看集群状态

   ```shell
   etcdctl \
     --ca-file=/opt/kubernetes/ssl/ca.pem \
     --cert-file=/opt/kubernetes/ssl/etcd.pem \
     --key-file=/opt/kubernetes/ssl/etcd-key.pem \
     --endpoints=https://114.67.166.108:2379,https://139.9.181.246:2379,https://106.55.0.70:2379 \
     cluster-health
   member 277fa914d3c04c05 is healthy: got healthy result from https://114.67.166.108:2379
   member 3807435c3e59004e is healthy: got healthy result from https://106.55.0.70:2379
   member c1e0956196d0f780 is healthy: got healthy result from https://139.9.181.246:2379
   cluster is healthy
   
   ```

   

## Master节点部署

### kube-apiserver部署

1. 创建证书请求文件

   ```shell
   cat > /opt/kubernetes/ssl/kube-apiserver-csr.json <<EOF
   {
     "CN": "kubernetes",
     "hosts": [
       "127.0.0.1",
       "192.168.0.3",
       "114.67.166.108",
       "10.254.0.1",
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

   `hosts`属性中需要包含所有Master节点的公网IP、内网IP和ClusterIP

2. 生成证书

   ```shell
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-api-server-csr.json | cfssljson -bare kube-apiserver
   ```

3. 查看，此时应该有如下文件

   ```shell
   [root@jd ssl]# pwd
   /opt/kubernetes/ssl
   [root@jd ssl]# ll
   total 100
   -rw-r--r-- 1 root root  378 May 25 20:23 ca-config.json
   -rw-r--r-- 1 root root  952 May 25 20:24 ca.csr
   -rw-r--r-- 1 root root  207 May 25 20:24 ca-csr.json
   -rw------- 1 root root 1679 May 25 20:24 ca-key.pem
   -rw-r--r-- 1 root root 1261 May 25 20:24 ca.pem
   -rw-r--r-- 1 root root 1017 May 25 20:26 etcd.csr
   -rw-r--r-- 1 root root  244 May 25 20:26 etcd-csr.json
   -rw------- 1 root root 1679 May 25 20:26 etcd-key.pem
   -rw-r--r-- 1 root root 1338 May 25 20:26 etcd.pem
   -rw-r--r-- 1 root root 1257 May 29 14:37 kube-apiserver.csr
   -rw-r--r-- 1 root root  460 May 29 14:35 kube-apiserver-csr.json
   -rw------- 1 root root 1675 May 29 14:37 kube-apiserver-key.pem
   -rw-r--r-- 1 root root 1574 May 29 14:37 kube-apiserver.pem
   
   ```

4. 生成Token配置文件，用于Node节点中kubelet加入集群验证

   * 生成Token

     ```shell
     head -c 16 /dev/urandom | od -An -t x | tr -d ' '
     dc7903bc34d75a708526369ceeb8250e
     ```

   * 创建token.csv

     ```shell
     cat > /opt/kubernetes/cfg/token.csv <<EOF
     dc7903bc34d75a708526369ceeb8250e,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
     EOF
     ```

     token.csv文件第一个参数标识一个tokenId，第二个参数标识一个用户，第三个标识用户组，第四个标识当前加入到k8s的角色

5. 创建加密配置文件

   * 生成秘钥key

     ```shell
     head -c 32 /dev/urandom | base64
     vQj908BrQAuTrxnLaqGbDrvfd2FJWsKiDdN0U5LfXgg=
     ```

   * 创建加密配置文件

     ```shell
     cat > /opt/kubernetes/cfg/encryption-config.yaml <<EOF
     kind: EncryptionConfig
     apiVersion: v1
     resources:
       - resources:
           - secrets
         providers:
           - aescbc:
               keys:
                 - name: key1
                   secret: vQj908BrQAuTrxnLaqGbDrvfd2FJWsKiDdN0U5LfXgg=
           - identity: {}
     EOF
     ```

6. 创建配置文件

   ```shell
   cat > /opt/kubernetes/cfg/kube-apiserver.conf <<EOF
   KUBE_APISERVER_ARGS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
     --anonymous-auth=false \\
     --experimental-encryption-provider-config=/opt/kubernetes/cfg/encryption-config.yaml \\
     --advertise-address=114.67.166.108 \\
     --bind-address=192.168.0.3 \\
     --insecure-port=0 \\
     --authorization-mode=Node,RBAC \\
     --runtime-config=api/all \\
     --enable-bootstrap-token-auth \\
     --token-auth-file=/opt/kubernetes/cfg/token.csv \\
     --service-cluster-ip-range=10.254.0.0/16 \\
     --tls-cert-file=/opt/kubernetes/ssl/kube-apiserver.pem \\
     --tls-private-key-file=/opt/kubernetes/ssl/kube-apiserver-key.pem \\
     --client-ca-file=/opt/kubernetes/ssl/ca.pem \\
     --kubelet-client-certificate=/opt/kubernetes/ssl/kube-apiserver.pem \\
     --kubelet-client-key=/opt/kubernetes/ssl/kube-apiserver-key.pem \\
     --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
     --etcd-cafile=/opt/kubernetes/ssl/ca.pem \\
     --etcd-certfile=/opt/kubernetes/ssl/etcd.pem \\
     --etcd-keyfile=/opt/kubernetes/ssl/etcd-key.pem \\
     --etcd-servers=https://114.67.166.108:2379,https://139.9.181.246:2379,https://106.55.0.70:2379 \\
     --enable-swagger-ui=true \\
     --allow-privileged=true \\
     --apiserver-count=2 \\
     --audit-log-maxage=30 \\
     --audit-log-maxbackup=3 \\
     --audit-log-maxsize=100 \\
     --audit-log-path=/var/log/kubernetes/kube-apiserver-audit.log \\
     --event-ttl=1h \\
     --alsologtostderr=true \\
     --logtostderr=false \\
     --log-dir=/var/log/kubernetes \\
     --v=2"
   EOF
   ```

   `--bind-address`：当前节点内网IP

   `--token-auth-file`：第4步生成的token.csv文件路径

7. 创建系统服务

   ```shell
   cat > /usr/lib/systemd/system/kube-apiserver.service <<EOF
   [Unit]
   Description=Kubernetes API Server
   Documentation=https://github.com/GoogleCloudPlatform/kubernetes
   After=network.target
    
   [Service]
   EnvironmentFile=-/opt/kubernetes/cfg/kube-apiserver.conf
   ExecStart=/opt/kubernetes/bin/kube-apiserver $KUBE_APISERVER_ARGS
   Restart=on-failure
   RestartSec=5
   Type=notify
   #User=k8s
   LimitNOFILE=65536
    
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

8. 启动

   ```shell
   systemctl daemon-reload
   
   systemctl start kube-apiserver.service
   
   systemctl enable kube-apiserver.service
   ```



###  部署kubectl

1. 创建证书请求文件

   ```shell
   cat > /opt/kubernetes/ssladmin-csr.json <<EOF
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

2. 生成证书

   ```shell
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
   ```

3. 查看，此时应该有如下文件

   ```
   [root@jd ssl]# pwd
   /opt/kubernetes/ssl
   [root@jd ssl]# ll
   total 100
   -rw-r--r-- 1 root root 1013 May 26 19:32 admin.csr
   -rw-r--r-- 1 root root  232 May 26 19:32 admin-csr.json
   -rw------- 1 root root 1675 May 26 19:32 admin-key.pem
   -rw-r--r-- 1 root root 1354 May 26 19:32 admin.pem
   -rw-r--r-- 1 root root  378 May 25 20:23 ca-config.json
   -rw-r--r-- 1 root root  952 May 25 20:24 ca.csr
   -rw-r--r-- 1 root root  207 May 25 20:24 ca-csr.json
   -rw------- 1 root root 1679 May 25 20:24 ca-key.pem
   -rw-r--r-- 1 root root 1261 May 25 20:24 ca.pem
   -rw-r--r-- 1 root root 1017 May 25 20:26 etcd.csr
   -rw-r--r-- 1 root root  244 May 25 20:26 etcd-csr.json
   -rw------- 1 root root 1679 May 25 20:26 etcd-key.pem
   -rw-r--r-- 1 root root 1338 May 25 20:26 etcd.pem
   -rw-r--r-- 1 root root 1257 May 29 14:37 kube-apiserver.csr
   -rw-r--r-- 1 root root  460 May 29 14:35 kube-apiserver-csr.json
   -rw------- 1 root root 1675 May 29 14:37 kube-apiserver-key.pem
   -rw-r--r-- 1 root root 1574 May 29 14:37 kube-apiserver.pem
   
   ```

4. 生成`admin.conf`配置文件并将其copy到用户目录下的`.kube`目录下

   ```
   cd /opt/kubernetes/cfg
   
   kubectl config set-cluster kubernetes \
       --certificate-authority=/opt/kubernetes/ssl/ca.pem \
       --embed-certs=true \
       --server=https://114.67.166.108:6443 \
       --kubeconfig=admin.conf
   	
   kubectl config set-credentials admin \
       --client-certificate=/opt/kubernetes/ssl/admin.pem \
       --client-key=/opt/kubernetes/ssl/admin-key.pem \
       --embed-certs=true \
       --kubeconfig=admin.conf
       
   kubectl config set-context kubernetes \
       --cluster=kubernetes \
       --user=admin \
       --kubeconfig=admin.conf
       
   kubectl config use-context kubernetes --kubeconfig=admin.conf
   
   cp admin.conf /root/.kube/config
   ```

   

### kube-controller-manager部署

1. 创建证书请求文件

   ```shell
   cat > /opt/kubernetes/ssl/kube-controller-manager-csr.json <<EOF
   {
       "CN": "system:kube-controller-manager",
       "key": {
           "algo": "rsa",
           "size": 2048
       },
       "hosts": [
           "127.0.0.1",
           "192.168.0.3",
   		   "114.67.166.108",
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
   EOF
   ```

   `hosts`属性中需要包含当前节点的内网IP和公网IP

   CN和O均为system:kube-controller-manager，kubernetes内置的ClusterRoleBindings system:kube-controller-manager赋予kube-controller-manager工作所需权限

2. 生成证书

   ```shell
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
   ```

3. 查看，此时应该有如下文件

   ```shell
   [root@jd ssl]# pwd
   /opt/kubernetes/ssl
   [root@jd ssl]# ll
   total 100
   -rw-r--r-- 1 root root 1013 May 26 19:32 admin.csr
   -rw-r--r-- 1 root root  232 May 26 19:32 admin-csr.json
   -rw------- 1 root root 1675 May 26 19:32 admin-key.pem
   -rw-r--r-- 1 root root 1354 May 26 19:32 admin.pem
   -rw-r--r-- 1 root root  378 May 25 20:23 ca-config.json
   -rw-r--r-- 1 root root  952 May 25 20:24 ca.csr
   -rw-r--r-- 1 root root  207 May 25 20:24 ca-csr.json
   -rw------- 1 root root 1679 May 25 20:24 ca-key.pem
   -rw-r--r-- 1 root root 1261 May 25 20:24 ca.pem
   -rw-r--r-- 1 root root 1017 May 25 20:26 etcd.csr
   -rw-r--r-- 1 root root  244 May 25 20:26 etcd-csr.json
   -rw------- 1 root root 1679 May 25 20:26 etcd-key.pem
   -rw-r--r-- 1 root root 1338 May 25 20:26 etcd.pem
   -rw-r--r-- 1 root root 1257 May 29 14:37 kube-apiserver.csr
   -rw-r--r-- 1 root root  460 May 29 14:35 kube-apiserver-csr.json
   -rw------- 1 root root 1675 May 29 14:37 kube-apiserver-key.pem
   -rw-r--r-- 1 root root 1574 May 29 14:37 kube-apiserver.pem
   -rw-r--r-- 1 root root 1196 May 26 19:49 kube-controller-manager.csr
   -rw-r--r-- 1 root root  452 May 26 19:48 kube-controller-manager-csr.json
   -rw------- 1 root root 1679 May 26 19:49 kube-controller-manager-key.pem
   -rw-r--r-- 1 root root 1521 May 26 19:49 kube-controller-manager.pem
   
   ```

4. 创建`kubeconfig`配置文件

   ```shell
   cd /opt/kubernetes/cfg
   
   kubectl config set-cluster kubernetes \
     --certificate-authority=/opt/kubernetes/ssl/ca.pem \
     --embed-certs=true \
     --server=https://114.67.166.108:6443 \
     --kubeconfig=kube-controller-manager.kubeconfig
     
   kubectl config set-credentials system:kube-controller-manager \
     --client-certificate=/opt/kubernetes/ssl/kube-controller-manager.pem \
     --client-key=/opt/kubernetes/ssl/kube-controller-manager-key.pem \
     --embed-certs=true \
     --kubeconfig=kube-controller-manager.kubeconfig
     
   kubectl config set-context system:kube-controller-manager \
     --cluster=kubernetes \
     --user=system:kube-controller-manager \
     --kubeconfig=kube-controller-manager.kubeconfig
     
   kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
   ```

5. 创建系统服务配置文件（如果在启动controller-manager时报错，尝试将配置文件里面的内容直接写入到系统服务`kube-controller-manager.service`中，之前在这里踩过坑，具体原因不详）

   ```shell
   cat > /opt/kubernetes/cfg/kube-controller-manager.conf <<EOf
   KUBE_CONTROLLER_MANAGER_ARGS="
     --profiling \\
     --cluster-name=kubernetes \\
     --controllers=*,bootstrapsigner,tokencleaner \\
     --kube-api-qps=1000 \\
     --kube-api-burst=2000 \\
     --leader-elect \\
     --use-service-account-credentials \\
     --concurrent-service-syncs=2 \\
     --tls-cert-file=/opt/kubernetes/ssl/kube-controller-manager.pem \\
     --tls-private-key-file=/opt/kubernetes/ssl/kube-controller-manager-key.pem \\
     --authentication-kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \\
     --client-ca-file=/opt/kubernetes/ssl/ca.pem \\
     --requestheader-allowed-names="" \\
     --requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \\
     --requestheader-extra-headers-prefix="X-Remote-Extra-" \\
     --requestheader-group-headers=X-Remote-Group \\
     --requestheader-username-headers=X-Remote-User \\
     --authorization-kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \\
     --cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
     --cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \\
     --experimental-cluster-signing-duration=876000h \\
     --horizontal-pod-autoscaler-sync-period=10s \\
     --concurrent-deployment-syncs=10 \\
     --concurrent-gc-syncs=30 \\
     --node-cidr-mask-size=24 \\
     --service-cluster-ip-range=10.254.0.0/16 \\
     --pod-eviction-timeout=6m \\
     --terminated-pod-gc-threshold=10000 \\
     --root-ca-file=/opt/kubernetes/ssl/ca.pem \\
     --service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
     --kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \\
     --logtostderr=false \\
     --log-dir=/var/log/kubernetes \\
     --v=2"
   EOF
   ```

6. 创建系统服务

   ```shell
   cat > /usr/lib/systemd/system/kube-controller-manager.service <<EOF
   [Unit]
   Description=Kubernetes Controller Manager
   Documentation=https://github.com/GoogleCloudPlatform/kubernetes
   
   [Service]
   EnvironmentFile=-/opt/kubernetes/cfg/kube-controller-manager.conf
   ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_ARGS
   Restart=on-failure
   RestartSec=5
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

   `service-cluster-ip-range`：指定 Service Cluster IP 网段，必须和 kube-apiserver 中的同名参数一致

7. 启动

   ```shell
   systemctl daemon-reload
   
   systemctl start kube-controller-manager.service
   
   systemctl enable kube-controller-manager.service
   ```

   

### kube-scheduler部署

1. 创建证书请求文件

   ```shell
   cat > /opt/kubernetes/ssl/kube-scheduler-csr.json <<EOF
   {
       "CN": "system:kube-scheduler",
       "hosts": [
         "127.0.0.1",
         "192.168.0.3",
         "114.67.166.108"
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

   `hosts`：需要指定当前节点内网IP和公网IP

2. 生成证书

   ```shell
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
   ```

3. 查看，此时应该有如下文件

   ```shell
   [root@jd ssl]# pwd
   /opt/kubernetes/ssl
   [root@jd ssl]# ll
   total 100
   -rw-r--r-- 1 root root 1013 May 26 19:32 admin.csr
   -rw-r--r-- 1 root root  232 May 26 19:32 admin-csr.json
   -rw------- 1 root root 1675 May 26 19:32 admin-key.pem
   -rw-r--r-- 1 root root 1354 May 26 19:32 admin.pem
   -rw-r--r-- 1 root root  378 May 25 20:23 ca-config.json
   -rw-r--r-- 1 root root  952 May 25 20:24 ca.csr
   -rw-r--r-- 1 root root  207 May 25 20:24 ca-csr.json
   -rw------- 1 root root 1679 May 25 20:24 ca-key.pem
   -rw-r--r-- 1 root root 1261 May 25 20:24 ca.pem
   -rw-r--r-- 1 root root 1017 May 25 20:26 etcd.csr
   -rw-r--r-- 1 root root  244 May 25 20:26 etcd-csr.json
   -rw------- 1 root root 1679 May 25 20:26 etcd-key.pem
   -rw-r--r-- 1 root root 1338 May 25 20:26 etcd.pem
   -rw-r--r-- 1 root root 1257 May 29 14:37 kube-apiserver.csr
   -rw-r--r-- 1 root root  460 May 29 14:35 kube-apiserver-csr.json
   -rw------- 1 root root 1675 May 29 14:37 kube-apiserver-key.pem
   -rw-r--r-- 1 root root 1574 May 29 14:37 kube-apiserver.pem
   -rw-r--r-- 1 root root 1196 May 26 19:49 kube-controller-manager.csr
   -rw-r--r-- 1 root root  452 May 26 19:48 kube-controller-manager-csr.json
   -rw------- 1 root root 1679 May 26 19:49 kube-controller-manager-key.pem
   -rw-r--r-- 1 root root 1521 May 26 19:49 kube-controller-manager.pem
   -rw-r--r-- 1 root root 1106 May 26 21:06 kube-scheduler.csr
   -rw-r--r-- 1 root root  357 May 26 21:05 kube-scheduler-csr.json
   -rw------- 1 root root 1679 May 26 21:06 kube-scheduler-key.pem
   -rw-r--r-- 1 root root 1432 May 26 21:06 kube-scheduler.pem
   
   ```

4. 创建`kubeconfig`配置文件

   ```shell
   cd /opt/kubernetes/cfg
   
   kubectl config set-cluster kubernetes \
     --certificate-authority=/opt/kubernetes/ssl/ca.pem \
     --embed-certs=true \
     --server=https://114.67.166.108:6443 \
     --kubeconfig=kube-scheduler.kubeconfig
   
   kubectl config set-credentials system:kube-scheduler \
     --client-certificate=/opt/kubernetes/ssl/kube-scheduler.pem \
     --client-key=/opt/kubernetes/ssl/kube-scheduler-key.pem \
     --embed-certs=true \
     --kubeconfig=kube-scheduler.kubeconfig
   
   kubectl config set-context system:kube-scheduler \
     --cluster=kubernetes \
     --user=system:kube-scheduler \
     --kubeconfig=kube-scheduler.kubeconfig
   
   kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig 
   ```

5. 创建系统服务配置文件

   ```shell
   cat > /opt/kubernetes/cfg/kube-scheduler.conf <<EOF
   KUBE_SCHEDULER_ARGS="--address=127.0.0.1 \\
     --kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \\
     --leader-elect=true \\
     --alsologtostderr=true \\
     --logtostderr=false \\
     --log-dir=/var/log/kubernetes \\
     --v=2"
   EOF
   ```

6. 创建系统服务

   ```shell
   cat > /usr/lib/systemd/system/kube-scheduler.service <<EOF
   [Unit]
   Description=Kubernetes Scheduler
   Documentation=https://github.com/GoogleCloudPlatform/kubernetes
   
   [Service]
   EnvironmentFile=-/opt/kubernetes/cfg/kube-scheduler.conf
   ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_ARGS
   Restart=on-failure
   RestartSec=5
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

7. 启动

   ```shell
   systemctl daemon-reload
   
   systemctl start kube-scheduler
   
   systemctl enable kube-scheduler
   ```

   

## Node节点部署

### docker安装

这里使用yum安装Docker，默认yum远程仓库中没有docker，所以需要添加repository，添加repository需要用到yum-config-manager，但是在使用yum-config-manager时报找不到该命令，所以需要安装yum-utils

1. 安装`yum-utils`

   ```shell
   yum install -y yum-utils device-mapper-persistent-data lvm2
   ```

2. 添加docker yum源

   ```shell
   yum-config-manager \
       --add-repo \
       https://download.docker.com/linux/centos/docker-ce.repo
   ```

3. 安装docker-ce及客户端

   ```shell
   yum install -y docker-ce-18.09.0-3.el7 docker-ce-cli-18.09.0-3.el7 containerd.io
   ```

4. 配置阿里云镜像加速（因为国内访问DockerHub比较慢）

   ```shell
   sudo mkdir -p /etc/docker
   
   sudo tee /etc/docker/daemon.json <<-'EOF'
   {
     "registry-mirrors": ["https://gqjyyepn.mirror.aliyuncs.com"]
   }
   EOF
   
   sudo systemctl daemon-reload
   
   sudo systemctl restart docker
   
   sudo systemctl enable docker
   ```

   

### flannel部署

Flannel的用于跨主机节点间docker容器的通信

1. 将分配给flannel的子网段写入到etcd中，避免多个flannel节点间ip冲突

   ```shell
   etcdctl \
     --ca-file=/opt/kubernetes/ssl/ca.pem \
     --cert-file=/opt/kubernetes/ssl/etcd.pem \
     --key-file=/opt/kubernetes/ssl/etcd-key.pem \
     --endpoints=https://114.67.166.108:2379,https://139.9.181.246:2379,https://106.55.0.70:2379 \
     set /coreos.com/network/config  '{ "Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'
   ```

2. 创建配置文件

   ```shell
   cat > /opt/kubernetes/cfg/flannel.conf <<EOF
   FLANNELD_PUBLIC_IP="139.9.181.246"
   FLANNELD_IFACE="eth0"
   
   FLANNEL_ETCD="--etcd-endpoints=https://114.67.166.108:2379,https://139.9.181.246:2379,https://106.55.0.70:2379"
   FLANNEL_ETCD_CAFILE="--etcd-cafile=/opt/kubernetes/ssl/ca.pem"
   FLANNEL_ETCD_CERTFILE="--etcd-certfile=/opt/kubernetes/ssl/etcd.pem"
   FLANNEL_ETCD_KEYFILE="--etcd-keyfile=/opt/kubernetes/ssl/etcd-key.pem"
   FLANNELD_IP_MASQ=true
   EOF
   ```

   **注意**

   `FLANNELD_PUBLIC_IP`：这里填写当前节点公网IP，官方支持的

   `FLANNELD_IFACE`：填写内网IP或者内网IP对应的网卡名称

3. 创建系统服务

   ```shell
   cat > /usr/lib/systemd/system/flanneld.service <<EOF
   [Unit]
   Description=Flanneld overlay address etc agent
   After=network-online.target network.target
   Before=docker.service
   
   [Service]
   Type=notify
   EnvironmentFile=-/opt/kubernetes/cfg/flannel.conf
   ExecStart=/bin/bash -c "GOMAXPROCS=1 /opt/kubernetes/bin/flanneld"
   ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
   
   Restart=on-failure
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

   新版本的`flannel`支持直接读取环境变量导入的配置，所以无需再后面添加参数

   `mk-docker-opts.sh`脚本将分配给flannel的Pod子网网段信息写入到`/run/flannel/docker`文件中，后续docker启动时使用这个文件中参数值设置docker0网桥

4. 配置docker使用flannel分配的子网

   * 修改docker.service文件

     ```shell
     vim /usr/lib/systemd/system/docker.service
     ```

     这里有三个地方需要修改

     ```shell
     [Unit]
     Description=Flanneld overlay address etc agent
     After=network-online.target network.target
     Before=docker.service
     
     
     [Service]
     Type=notify
     EnvironmentFile=-/opt/kubernetes/cfg/flannel.conf
     ExecStart=/bin/bash -c "GOMAXPROCS=1 /opt/kubernetes/bin/flanneld"
     ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
     
     
     Restart=on-failure
     
     
     [Install]
     WantedBy=multi-user.target
     [root@hw cfg]# cat /usr/lib/systemd/system/docker.service
     [Unit]
     Description=Docker Application Container Engine
     Documentation=https://docs.docker.com
     BindsTo=containerd.service
     After=network-online.target firewalld.service
     Wants=network-online.target
     Requires=flanneld.service
     
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
     
     # Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
     # Both the old, and new location are accepted by systemd 229 and up, so using the old location
     # to make them work for either version of systemd.
     StartLimitBurst=3
     
     # Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
     # Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
     # this option work for either version of systemd.
     StartLimitInterval=60s
     
     # Having non-zero Limit*s causes performance problems due to accounting overhead
     # in the kernel. We recommend using cgroups to do container-local accounting.
     LimitNOFILE=infinity
     LimitNPROC=infinity
     LimitCORE=infinity
     
     # Comment TasksMax if your systemd version does not supports it.
     # Only systemd 226 and above support this option.
     TasksMax=infinity
     
     # set delegate yes so that systemd does not reset the cgroups of docker containers
     Delegate=yes
     
     # kill only the docker process, not all processes in the cgroup
     KillMode=process
     
     [Install]
     WantedBy=multi-user.target
     
     ```

     在Unit段中的After后面添加flannel.service参数

     在Wants下面添加`Requires=flanneld.service`

     在Service段中Type后面添加`EnvironmentFile=-/run/flannel/docker`

     在ExecStart后面添加`$DOCKER_NETWORK_OPTIONS`参数

5. 启动服务

   ```shell
   systemctl daemon-reload
   
   systemctl start flanneld.service
   
   systemctl enable flanneld.service
   
   systemctl restart docker.service
   ```

   查看

   ```shell
   ip a
   # 省略其他网卡信息
   3: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
       link/ether 72:c6:c5:14:c4:07 brd ff:ff:ff:ff:ff:ff
       inet 10.254.12.0/32 scope global flannel.1
          valid_lft forever preferred_lft forever
       inet6 fe80::70c6:c5ff:fe14:c407/64 scope link 
          valid_lft forever preferred_lft forever
   4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
       link/ether 02:42:34:3d:98:65 brd ff:ff:ff:ff:ff:ff
       inet 10.254.12.1/24 brd 10.254.12.255 scope global docker0
          valid_lft forever preferred_lft forever
       inet6 fe80::42:34ff:fe3d:9865/64 scope link 
          valid_lft forever preferred_lft forever
   
   ```

   如果发现flannel和docker0出于同一个网段则证明部署成功

### kubelet部署

1. 为kubelet创建一个bootstrap.kubeconfig文件，这里先在master节点上执行，然后拷贝到worker节点，因为只有master节点上安装了kubectl

   ```shell
   cd /opt/kubernetes/cfg
   
   kubectl config set-cluster kubernetes \
         --certificate-authority=/opt/kubernetes/ssl/ca.pem \
         --embed-certs=true \
         --server=https://114.67.166.108:6443 \
         --kubeconfig=kubelet-bootstrap.kubeconfig
   
   # 设置客户端认证参数
   kubectl config set-credentials kubelet-bootstrap \
         --token=7e5702c8dbf9843aa831b5d6aa08e722 \
         --kubeconfig=kubelet-bootstrap.kubeconfig
   
   # 设置上下文参数
   kubectl config set-context default \
         --cluster=kubernetes \
         --user=kubelet-bootstrap \
         --kubeconfig=kubelet-bootstrap.kubeconfig
   
   # 设置默认上下文
   kubectl config use-context default --kubeconfig=kubelet-bootstrap.kubeconfig
   
   scp kubelet-bootstrap.kubeconfig root@hw:/opt/kubernetes/cfg
   
   scp kubelet-bootstrap.kubeconfig root@tx:/opt/kubernetes/cfg
   ```

2. 创建kubelet.config.json文件

   ```shell
   cat > /opt/kubernetes/cfg/kubelet.config <<EOF
   kind: KubeletConfiguration
   apiVersion: kubelet.config.k8s.io/v1beta1
   address: 172.16.0.5
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
   
   scp kubelet.config root@hw:/opt/kubernetes/cfg
   
   scp kubelet.config root@tx:/opt/kubernetes/cfg
   ```

3. 创建系统服务配置文件

   ```shell
   cat > /opt/kubernetes/cfg/kubelet.conf <<EOF
   KUBELET_ARGS="--bootstrap-kubeconfig=/opt/kubernetes/cfg/kubelet-bootstrap.kubeconfig \\   
   --cert-dir=/opt/kubernetes/ssl \\
   --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
   --config=/opt/kubernetes/cfg/kubelet.config \\
   --pod-infra-container-image=k8s.gcr.io/pause-amd64:3.1 \\
   --allow-privileged=true \\
   --alsologtostderr=true \\
   --logtostderr=false \\
   --log-dir=/var/log/kubernetes \\
   --v=2"
   EOF
   ```

   `kubelet.config`会自动生成

4. 创建系统服务

   ```shell
   cat > /usr/lib/systemd/system/kubelet.service <<EOF
   [Unit]
   Description=Kubernetes Kubelet
   Documentation=https://github.com/GoogleCloudPlatform/kubernetes
   After=docker.service
   Requires=docker.service
   
   [Service]
   EnvironmentFile=-/opt/kubernetes/cfg/kubelet.conf
   ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_ARGS
     
   Restart=on-failure
   RestartSec=5
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

5. 由于pause镜像默认拉取需要科学上网，所有这里通过国内下载打标签的方式预先下载好镜像

   ```shell
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

   可以按需下载，也可以直接执行以上脚本拉取所有k8s所需要的镜像

6. 将kubelet-bootstrap用户绑定到系统集群角色（当前操作在master上执行）

   ```shell
   kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
   ```

7. 启动

   ```shell
   mkdir -p /var/lib/kubelet
   
   systemctl daemon-reload
   
   systemctl start kubelet.service
   
   systemctl enable kubelet.service
   ```

8. 在Master节点上审批csr请求

   ```shell
   kubectl get csr
   NAME                                                   AGE   REQUESTOR           CONDITION
   node-csr-C4O9_KIek83fXKlhPjsW37KxpzBGl6CSspvsDEiBsPc   18s   kubelet-bootstrap   Pending
   node-csr-Ow3aKEezOFC3bGIerrIu_olmsKEb02GNECffcfOYYZY   18s   kubelet-bootstrap   Pending
   
   kubectl certificate approve node-csr-C4O9_KIek83fXKlhPjsW37KxpzBGl6CSspvsDEiBsPc
   
   kubectl certificate approve node-csr-Ow3aKEezOFC3bGIerrIu_olmsKEb02GNECffcfOYYZY
   ```

   

### kube-proxy部署

1. 创建证书请求文件（当前操作在Master上执行，完成后分发到Node节点）

   ```shell
   cat > /ope/kubernetes/ssl/kube-proxy-csr.json <<EOF
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

2. 生成证书

   ```shell
   cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
   
   scp kube-proxy*.* root:hw:/ope/kubernetes/ssl
   
   scp kube-proxy*.* root:tx:/ope/kubernetes/ssl
   ```

3. 创建`kubeconfig`配置文件

   ```shell
   cd /ope/kubernetes/cfg
   
   kubectl config set-cluster kubernetes \
     --certificate-authority=/opt/kubernetes/ssl/ca.pem \
     --embed-certs=true \
     --server=https://114.67.166.108:6443 \
     --kubeconfig=kube-proxy.kubeconfig
     
   kubectl config set-credentials kube-proxy \
     --client-certificate=/opt/kubernetes/ssl/kube-proxy.pem \
     --client-key=/opt/kubernetes/ssl/kube-proxy-key.pem \
     --embed-certs=true \
     --kubeconfig=kube-proxy.kubeconfig
     
   kubectl config set-context default \
     --cluster=kubernetes \
     --user=kube-proxy \
     --kubeconfig=kube-proxy.kubeconfig
     
   kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
   
   scp kube-proxy.kubeconfig root:hw:/ope/kubernetes/cfg
   
   scp kube-proxy.kubeconfig root:tx:/ope/kubernetes/cfg
   ```

4. 创建系统服务配置文件

   ```shell
   cat > /ope/kubernetes/cfg/kube-proxy.conf <<EOF
   KUBE_PROXY_ARGS="--config=/opt/kubernetes/cfg/kube-proxy-config.yaml \\
   --alsologtostderr=true \\
   --logtostderr=false \\
   --log-dir=/var/log/kubernetes \\
   --v=2"
   EOF
   ```

5. 创建系统服务

   ```shell
   cat > /usr/lib/systemd/system/kube-proxy.service <<EOF
   [Unit]
   Description=Kubernetes Kube-Proxy Server
   Documentation=https://github.com/GoogleCloudPlatform/kubernetes
   After=network.target
   
   [Service]
   EnvironmentFile=-/opt/kubernetes/cfg/kube-proxy.conf
   ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_ARGS
   Restart=on-failure
   RestartSec=5
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

6. 启动

   ```shell
   systemctl daemon-reload
   
   systemctl enable kube-proxy 
   
   systemctl start kube-proxy
   ```

   

## 自动补全

1. 安装`bash-completion`

   ```shell
   yum install bash-completion
   ```

2. 配置

   ```shell
   source /usr/share/bash-completion/bash_completion
   
   source <(kubectl completion bash)
   
   echo "source <(kubectl completion bash)" >> ~/.bashrc
   ```

   



## 参考文献

> https://blog.csdn.net/chen645800876/article/details/105279648/

> https://gitee.com/admxj/kubernetes-ha-binary

