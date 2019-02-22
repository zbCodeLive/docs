# Kubernetes搭建

## 软件安装一览图

ip | 作用 | 服务
---|---|---
192.168.2.98 | loadbalance    |keepalived,nginx
192.168.2.99 | loadbalance    |keepalived,nginx
192.168.2.100| k8s的master节点|kube-apiserver,kube-controller-manager,kube-scheduler
192.168.2.101| k8s的master节点|kube-apiserver,kube-controller-manager,kube-scheduler
192.168.2.102| k8s的noed节点  |kubelet,kube-proxy
192.168.2.103| k8s的node节点  |kubelet,kube-proxy
192.168.2.253| vip            |


## 安装etcd

### 自签etcd证书

1. 安装cfssl与cfssljson
```
curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /usr/local/bin/cfssl
curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /usr/local/bin/cfssljson
curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o /usr/local/bin/cfssl-certinfo
chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson /usr/local/bin/cfssl-certinfo
```

2. 新建etcd自签证书配置文件 

```
mkdir -p /opt/etcd-cert
touch /opt/etcd-cert/etcd-cert.sh
```

```
etcd-cert.sh内容如下：

cat > ca-config.json <<EOF
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

cat > ca-csr.json <<EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

#-----------------------

cat > server-csr.json <<EOF
{
    "CN": "etcd",
    "hosts": [
    "192.168.2.101",
    "192.168.2.102",
    "192.168.2.103"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server

```

3. 生成etcd自签证书
```
bash etcd-cert.sh
```

4. 生成证书包含如下文件
```
ca-config.json, ca-config.json, ca-csr.json, ca-key.pem, ca.pem,
server.csr, server-csr.json, server-key.pem, server.pem
```

### 安装etcd cluster
1. 下载etcd安装包地址:  
https://github.com/etcd-io/etcd/releases

2. 新建etcd软件存放目录，与安装包存放路径：
```
mkdir -p /data/soft
mkdir -p /data/server
```

3. 解压etcd-v3.3.10-linux-amd64.tar.gz
```
tar -zxvf /data/soft/etcd-v3.3.10-linux-amd64.tar.gz -C /data/server
```

4. 新建etcd目录，并拷贝etcd执行文件与etcd证书文件
```
mkdir -p /opt/etcd/{bin,cfg,ssl}
cp /data/server/etcd-v3.3.10-linux-amd64/{etcd,etcdctl} /opt/etcd/bin
cp /opt/etcd-cert/* /opt/etcd/ssl/
```

5. 生成etcd配置文件,并安装etcd
```
vi /opt/etcd/etcd.sh
```
```
etcd.sh

#!/bin/bash
# example: ./etcd.sh etcd01 192.168.1.10 etcd02=https://192.168.1.11:2380,etcd03=https://192.168.1.12:2380

ETCD_NAME=$1
ETCD_IP=$2
ETCD_CLUSTER=$3

WORK_DIR=/opt/etcd

cat <<EOF >$WORK_DIR/cfg/etcd
#[Member]
ETCD_NAME="${ETCD_NAME}"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://${ETCD_IP}:2380"
ETCD_LISTEN_CLIENT_URLS="https://${ETCD_IP}:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://${ETCD_IP}:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://${ETCD_IP}:2379"
ETCD_INITIAL_CLUSTER="${ETCD_NAME}=https://${ETCD_IP}:2380,${ETCD_CLUSTER}"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF

cat <<EOF >/usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=${WORK_DIR}/cfg/etcd
ExecStart=${WORK_DIR}/bin/etcd \
--name=\${ETCD_NAME} \
--data-dir=\${ETCD_DATA_DIR} \
--listen-peer-urls=\${ETCD_LISTEN_PEER_URLS} \
--listen-client-urls=\${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
--advertise-client-urls=\${ETCD_ADVERTISE_CLIENT_URLS} \
--initial-advertise-peer-urls=\${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
--initial-cluster=\${ETCD_INITIAL_CLUSTER} \
--initial-cluster-token=\${ETCD_INITIAL_CLUSTER_TOKEN} \
--initial-cluster-state=new \
--cert-file=${WORK_DIR}/ssl/server.pem \
--key-file=${WORK_DIR}/ssl/server-key.pem \
--peer-cert-file=${WORK_DIR}/ssl/server.pem \
--peer-key-file=${WORK_DIR}/ssl/server-key.pem \
--trusted-ca-file=${WORK_DIR}/ssl/ca.pem \
--peer-trusted-ca-file=${WORK_DIR}/ssl/ca.pem
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable etcd
systemctl restart etcd

```

```
执行etcd.sh
bash /opt/etcd/etcd.sh etcd01 192.168.2.101 etcd02=https://192.168.2.102:2380,etcd03=https://192.168.2.103:2380
```

6. 安装etcd其余两个节点
```
scp -r /opt root@192.168.2.102:/
在192.168.2.102节点上执行：
bash /opt/etcd/etcd.sh etcd02 192.168.2.102 etcd01=https://192.168.2.101:2380,etcd03=https://192.168.2.103:2380

scp -r /opt root@192.168.2.103:/
在192.168.2.103节点上执行：
bash /opt/etcd/etcd.sh etcd03 192.168.2.103 etcd01=https://192.168.2.101:2380,etcd02=https://192.168.2.102:2380
```
ps：执行上述命令后需要Ctrl+C阻断挂起进程

7. 检查etcd cluster集群状态
```
/opt/etcd/bin/etcdctl \
  --ca-file=/opt/etcd/ssl/ca.pem \
  --cert-file=/opt/etcd/ssl/server.pem \
  --key-file=/opt/etcd/ssl/server-key.pem \
  cluster-health
```

## 安装Docker(node节点)
```
yum -y install yum-utils device-mapper-persistent-data lvm2 container-selinux libcgroup pigz libseccomp
rpm -ivh libltdl7-2.4.2-alt7.x86_64.rpm
rpm -ivh docker-ce-18.03.1.ce-1.el7.centos.x86_64.rpm
```

## 安装flanneld

ps:flannel网络需要在docker
1. 写入分配的子网段到etcd，供flanneld使用
```
/opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem --endpoint="https://192.168.2.101:2379,https://192.168.2.102:2379,https://192.168.2.103:2379" set /coreos.com/network/config '{"Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'
```

2. 新建kubernetes目录
```
mkdir -p /opt/kubernetes/{bin,cfg,ssl}
```

3. 解压flannel
```
tar -zxvf /data/soft/flannel-v0.10.0-linux-amd64.tar.gz -C /opt/kubernetes/bin/
```

4. 生成flannel配置文件，并启动flannel
```
vi /opt/kubernetes/flannel.sh

#!/bin/bash

ETCD_ENDPOINTS=${1:-"http://127.0.0.1:2379"}

cat <<EOF >/opt/kubernetes/cfg/flanneld

FLANNEL_OPTIONS="--etcd-endpoints=${ETCD_ENDPOINTS} \
-etcd-cafile=/opt/etcd/ssl/ca.pem \
-etcd-certfile=/opt/etcd/ssl/server.pem \
-etcd-keyfile=/opt/etcd/ssl/server-key.pem"

EOF

cat <<EOF >/usr/lib/systemd/system/flanneld.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network-online.target network.target
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/opt/kubernetes/cfg/flanneld
ExecStart=/opt/kubernetes/bin/flanneld --ip-masq \$FLANNEL_OPTIONS
ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env
Restart=on-failure

[Install]
WantedBy=multi-user.target

EOF

systemctl daemon-reload
systemctl enable flanneld
systemctl restart flanneld

```

```
bash /opt/kubernetes/flannel.sh
```

5. 配置docker使用flannel网络

```
vi /usr/lib/systemd/system/docker.service

ExecStart=/usr/bin/dockerd
修改为
EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
```

/usr/lib/systemd/system/docker.service
```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

6. 重启docker.service服务
```
systemctl daemon-reload
systemctl restart docker
```

## kubernetes master节点安装

### 安装kube-apiserver
1. 主节点上新建kukbernetes目录
```
mkdir -p /opt/kubernetes/{bin,cfg,ssl}
```

2. 自签apiserver证书

vi /opt/kubernetes/ssl/k8s-cert.sh  
server-csr.json中"hosts"需要写入master_ip,lb_ip,vip
```
cat > ca-config.json <<EOF
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

cat > ca-csr.json <<EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
      	    "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

#-----------------------

cat > server-csr.json <<EOF
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "192.168.2.98",
      "192.168.2.99",
      "192.168.2.100",
      "192.168.2.101",
      "192.168.2.253",
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
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server

#-----------------------

cat > admin-csr.json <<EOF
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
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

#-----------------------

cat > kube-proxy-csr.json <<EOF
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
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

```

执行k8s-cert.sh,生成apiserver自签证书
```
cd /opt/kubernetes/ssl
bash /opt/kubernetes/ssl/k8s-cert.sh
```

3. 新建apiserver.sh，并安装kube-apiserver  

解压并拷贝kubectl等执行文件
```
tar -zxvf /data/soft/kubernetes-server-linux-amd64.tar.gz -C /data/server/
cp /data/server/kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kubectl,kube-scheduler} /opt/kubernetes/bin/
```
vi /opt/kubernetes/apiserver.sh
```
#!/bin/bash

MASTER_ADDRESS=$1
ETCD_SERVERS=$2

cat <<EOF >/opt/kubernetes/cfg/kube-apiserver

KUBE_APISERVER_OPTS="--logtostderr=true \\
--v=4 \\
--etcd-servers=${ETCD_SERVERS} \\
--bind-address=${MASTER_ADDRESS} \\
--secure-port=6443 \\
--advertise-address=${MASTER_ADDRESS} \\
--allow-privileged=true \\
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \\
--authorization-mode=RBAC,Node \\
--kubelet-https=true \\
--enable-bootstrap-token-auth \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-50000 \\
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \\
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--etcd-cafile=/opt/etcd/ssl/ca.pem \\
--etcd-certfile=/opt/etcd/ssl/server.pem \\
--etcd-keyfile=/opt/etcd/ssl/server-key.pem"

EOF

cat <<EOF >/usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kube-apiserver
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-apiserver
systemctl restart kube-apiserver
```

生成/opt/kubernetes/cfg/token.csv文件
```
BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')

cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

执行apiserver.sh
```
bash /opt/kubernetes/apiserver.sh 192.168.2.101 https://192.168.2.101:2379,https://192.168.2.102:2379,https://192.168.2.103:2379
```

查看kube-apiserver是否启动成功
```
[root@CentOS7 kubernetes]# ps -aux | grep kube
root       376  121  3.2 344992 257100 ?       Ssl  18:29   0:08 /opt/kubernetes/bin/kube-apiserver --logtostderr=true --v=4 --etcd-servers=https://192.168.2.101:2379,https://192.168.2.102:2379,https://192.168.2.103:2379 --bind-address=192.168.2.101 --secure-port=6443 --advertise-address=192.168.2.101 --allow-privileged=true --service-cluster-ip-range=10.0.0.0/24 --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction --authorization-mode=RBAC,Node --kubelet-https=true --enable-bootstrap-token-auth --token-auth-file=/opt/kubernetes/cfg/token.csv --service-node-port-range=30000-50000 --tls-cert-file=/opt/kubernetes/ssl/server.pem --tls-private-key-file=/opt/kubernetes/ssl/server-key.pem --client-ca-file=/optkubernetes/ssl/ca.pem --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem --etcd-cafile=/opt/etcd/ssl/ca.pem --etcd-certfile=/opt/etcd/ssl/server.pem --etcd-keyfile=/opt/etcd/ssl/server-key.pem
```

### 安装kube-controller-manager

1. vi /opt/kubernetes/controller-manager.sh
```
#!/bin/bash

MASTER_ADDRESS=$1

cat <<EOF >/opt/kubernetes/cfg/kube-controller-manager


KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=true \\
--v=4 \\
--master=${MASTER_ADDRESS}:8080 \\
--leader-elect=true \\
--address=127.0.0.1 \\
--service-cluster-ip-range=10.0.0.0/24 \\
--cluster-name=kubernetes \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--experimental-cluster-signing-duration=87600h0m0s"

EOF

cat <<EOF >/usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kube-controller-manager
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl restart kube-controller-manager
```

2. 执行controller-manager.sh
```
bash /opt/kubernetes/controller-manager.sh 127.0.0.1
```

### 安装kube-scheduler

1. vi /opt/kubernetes/scheduler.sh
```
#!/bin/bash

MASTER_ADDRESS=$1

cat <<EOF >/opt/kubernetes/cfg/kube-scheduler

KUBE_SCHEDULER_OPTS="--logtostderr=true \\
--v=4 \\
--master=${MASTER_ADDRESS}:8080 \\
--leader-elect"

EOF

cat <<EOF >/usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kube-scheduler
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-scheduler
systemctl restart kube-scheduler

```

2. 执行controller-manager.sh
```
bash /opt/kubernetes/scheduler.sh 127.0.0.1
```

## kubernetes Node节点安装

### master节点上生成Node节点的配置文件

1. 创建角色(在master节点上执行)
```
export PATH=/opt/kubernetes/bin:$PATH

kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

2. 在master节点上生成node节点的配置文件  
vi /opt/kubernetes/kubeconfig.sh
```
#!/bin/bash

BOOTSTRAP_TOKEN=$(cat /opt/kubernetes/cfg/token.csv | awk -F ',' '{print $1}')

APISERVER=$1
SSL_DIR=$2

# 创建kubelet bootstrapping kubeconfig 
export KUBE_APISERVER="https://$APISERVER:6443"

# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=$SSL_DIR/ca.pem \
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

#----------------------

# 创建kube-proxy kubeconfig文件

kubectl config set-cluster kubernetes \
  --certificate-authority=$SSL_DIR/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
  --client-certificate=$SSL_DIR/kube-proxy.pem \
  --client-key=$SSL_DIR/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

3. 执行kubeconfig.sh
```
cd /opt/kubernetes/cfg

bash /opt/kubernetes/kubeconfig.sh 192.168.2.101 /opt/kubernetes/ssl
```

4. 拷贝master上生成的bootstrap.kubeconfig，kube-proxy.kubeconfig到node节点上
```
scp /opt/kubernetes/cfg/{bootstrap.kubeconfig,kube-proxy.kubeconfig} root@192.168.2.102:/opt/kubernetes/cfg/
scp /opt/kubernetes/cfg/{bootstrap.kubeconfig,kube-proxy.kubeconfig} root@192.168.2.103:/opt/kubernetes/cfg/
```

### 安装kubelet
1. 从master节点上拷贝kubelet，kube-proxy到node节点上
```
scp /data/server/kubernetes/server/bin/{kubelet,kube-proxy} root@192.168.2.102:/opt/kubernetes/bin/

scp /data/server/kubernetes/server/bin/{kubelet,kube-proxy} root@192.168.2.103:/opt/kubernetes/bin/
```

2. vi /opt/kubernetes/kubelet.sh
```
#!/bin/bash

NODE_ADDRESS=$1
DNS_SERVER_IP=${2:-"10.0.0.2"}

cat <<EOF >/opt/kubernetes/cfg/kubelet

KUBELET_OPTS="--logtostderr=true \\
--v=4 \\
--hostname-override=${NODE_ADDRESS} \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet.config \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"

EOF

cat <<EOF >/opt/kubernetes/cfg/kubelet.config

kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: ${NODE_ADDRESS}
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- ${DNS_SERVER_IP} 
clusterDomain: cluster.local.
failSwapOn: false
authentication:
  anonymous:
    enabled: true
EOF

cat <<EOF >/usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
KillMode=process

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kubelet
systemctl restart kubelet

```

2. 执行kubelet.sh
```
bash /opt/kubernetes/kubelet.sh 192.168.2.102
```

### 安装kube-proxy

1. vi /opt/kubernetes/kube-proxy.sh
```
#!/bin/bash

NODE_ADDRESS=$1

cat <<EOF >/opt/kubernetes/cfg/kube-proxy

KUBE_PROXY_OPTS="--logtostderr=true \\
--v=4 \\
--hostname-override=${NODE_ADDRESS} \\
--cluster-cidr=10.0.0.0/24 \\
--proxy-mode=ipvs \\
--masquerade-all=true \\
--kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig"

EOF

cat <<EOF >/usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kube-proxy
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-proxy
systemctl restart kube-proxy

```

2. 执行kube-proxy.sh
```
bash /opt/kubernetes/kube-proxy.sh 192.168.2.102
```

3. node节点加入集群
```
kubectl get csr
```
返回信息如下：
```
[root@CentOS7 kubernetes]# kubectl get csr
NAME                                                   AGE     REQUESTOR           CONDITION
node-csr-H1SSHMpjA_Qkt6FYb75OzVEq9BKW-H_uVqJWY49rMds   2m43s   kubelet-bootstrap   Pending
node-csr-ufyQa_RVw8EkZNEndb5sGX5TaSxTvGRAjuaIHk71M2U   13h     kubelet-bootstrap   Pending
```
将节点加入认证：
```
kubectl certificate approve node-csr-H1SSHMpjA_Qkt6FYb75OzVEq9BKW-H_uVqJWY49rMds
kubectl certificate approve node-csr-ufyQa_RVw8EkZNEndb5sGX5TaSxTvGRAjuaIHk71M2U
```
验证node节点是否加入master
```
kubectl get node
```
返回信息如下：
```
[root@CentOS7 kubernetes]# kubectl get node
NAME            STATUS     ROLES    AGE   VERSION
192.168.2.102   NotReady   <none>   9s    v1.12.0
192.168.2.103   Ready      <none>   32s   v1.12.0
```

## 部署第二个Master

拷贝kubernetes与service到第二个master  
(需要在192.168.2.100上新建/opt目录)
```
在第一个master上执行：
scp -r /opt root@192.168.2.100:/
scp /usr/lib/systemd/system/{kube-apiserver,kube-controller-manager,kube-scheduler}.service root@192.168.2.100:/usr/lib/systemd/system/

在第二个master上执行：
修改/opt/kubernetes/cfg/kube-apiserver为
"
KUBE_APISERVER_OPTS="--logtostderr=true \
--v=4 \
--etcd-servers=https://192.168.2.101:2379,https://192.168.2.102:2379,https://192.168.2.103:2379 \
--bind-address=192.168.2.100 \
--secure-port=6443 \
--advertise-address=192.168.2.100 \
--allow-privileged=true \
--service-cluster-ip-range=10.0.0.0/24 \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
--authorization-mode=RBAC,Node \
--kubelet-https=true \
--enable-bootstrap-token-auth \
--token-auth-file=/opt/kubernetes/cfg/token.csv \
--service-node-port-range=30000-50000 \
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \
--client-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
--etcd-cafile=/opt/etcd/ssl/ca.pem \
--etcd-certfile=/opt/etcd/ssl/server.pem \
--etcd-keyfile=/opt/etcd/ssl/server-key.pem"
"


systemctl daemon-reload
systemctl enable kube-apiserver && systemctl restart kube-apiserver
systemctl enable kube-controller-manager && systemctl restart kube-controller-manager
systemctl enable kube-scheduler && systemctl restart kube-scheduler
```

## Nginx安装

（在98，99两台机器上安装nginx作为lb）
1. 配置nginx的yum源
```
cat > /etc/yum.repos.d/nginx.repo << EOF
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/\$basearch/
gpgcheck=0
EOF
```

2. 安装nginx
```
yum -y install nginx
```

3. 修改/etc/nginx/nginx.conf
```
添加如下内容：
stream {

   log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';
    access_log  /var/log/nginx/k8s-access.log  main;

    upstream k8s-apiserver {
        server 192.168.2.100:6443;
        server 192.168.2.101:6443;
    }

    server {
    	listen 6443;
    	proxy_pass k8s-apiserver;
    }
}
```

4. 启动nginx
```
systemctl start nginx
```

## KeepAlived安装

1. 安装keepalived
```
yum -y install keepalived
```

2. 修改/etc/keepalived/keepalived.conf
```
! Configuration File for keepalived 
 
global_defs { 
   # 接收邮件地址 
   notification_email { 
     acassen@firewall.loc 
     failover@firewall.loc 
     sysadmin@firewall.loc 
   } 
   # 邮件发送地址 
   notification_email_from Alexandre.Cassen@firewall.loc  
   smtp_server 127.0.0.1 
   smtp_connect_timeout 30 
   router_id NGINX_MASTER 
} 

vrrp_script check_nginx {
    script "/usr/local/nginx/sbin/check_nginx.sh"
}

vrrp_instance VI_1 { 
    state MASTER 
    interface ens160
    virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的 
    priority 100    # 优先级，备服务器设置 90 
    advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒 
    authentication { 
        auth_type PASS      
        auth_pass 1111 
    }  
    virtual_ipaddress { 
        192.168.2.98/24 
    } 
    track_script {
        check_nginx
    } 
}

```
PS：根据实际情况修改state,interface,priority,virtual_ipaddress配置项

3. 设置nginx健康检查脚本
```
mkdir -p /usr/local/nginx/sbin/
vi /usr/local/nginx/sbin/check_nginx.sh
chmod +x /usr/local/nginx/sbin/check_nginx.sh


check_nginx.sh脚本如下：
count=$(ps -ef |grep nginx |egrep -cv "grep|$$")

if [ "$count" -eq 0 ];then
    systemctl stop keepalived
fi
```

4. 启动keepalived
```
systemctl start keepalived
```

PS:Keepalived启动后，修改node配置文件bootstrap.kubeconfig,kubelet.kubeconfig,kube-proxy.kubeconfig中连接master节点的ip192.168.2.101为vip192.168.2.253

## 搭建kubernetes UI
1. kubelet授权
```
kubectl create clusterrolebinding clulster-system-anonymous --clusterrole=cluster-admin --user=system:anonymous
```

2. 构建kubernetes UI
```
kubectl create -f dashboard-rbac.yaml
kubectl create -f dashboard-secret.yaml
kubectl create -f dashboard-configmap.yaml
kubectl create -f dashboard-controller.yaml
kubectl create -f dashboard-service.yaml
kubectl create -f k8s-admin.yaml
```

3. 获取令牌
```
[root@CentOS7 dashboard]# kubectl get secret -n kube-system
NAME                               TYPE                                  DATA   AGE
dashboard-admin-token-p44vg        kubernetes.io/service-account-token   3      6s
default-token-rz2kc                kubernetes.io/service-account-token   3      21h
kubernetes-dashboard-certs         Opaque                                0      59s
kubernetes-dashboard-key-holder    Opaque                                2      59s
kubernetes-dashboard-token-tb8l4   kubernetes.io/service-account-token   3      48s
[root@CentOS7 dashboard]# kubectl describe secret dashboard-admin-token-p44vg -n kube-system
Name:         dashboard-admin-token-p44vg
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: 73845214-fac0-11e8-9e23-005056b61cef

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1359 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tcDQ0dmciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNzM4NDUyMTQtZmFjMC0xMWU4LTllMjMtMDA1MDU2YjYxY2VmIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.egKIr4VR1hPkJdxcmxZ4WFpi3PbWayHNxXCE2gq2eX0U-IQBkYikdKuQJvUd1GEzHt1xHmD9thZvKbNTBfBlNKvLse55nKZ6fXLeHxtcVgv7Df8L09xw_bV35hFxeLRTuYF785Y7X2rUf7DRqNIO28HFagOcp7QXp61GdTIu0f__uObEs2NlV-SFflPuQpBAIZxt0iS-T7gVR6uUNdfW7C0rT6af6BECNAdPR4dWHhyUd2HOcTmu4v-9Y8gEonCIvcwhZ_7F43Z-M4Ayi4XhWdSYw1N6elaZewxsuEUfG_EgDLAkLqa1EAmhnQTS4TEqbWRehlQ2O6qhrjbH-_vx0g
[root@CentOS7 dashboard]# kubectl get pod,svc -n kube-system
NAME                                        READY   STATUS    RESTARTS   AGE
pod/kubernetes-dashboard-65f974f565-ft2qg   1/1     Running   0          102s

NAME                           TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
service/kubernetes-dashboard   NodePort   10.0.0.206   <none>        443:30001/TCP   97s
```
