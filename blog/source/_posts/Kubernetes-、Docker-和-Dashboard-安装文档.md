---
title: Kubernetes 、Docker  和 Dashboard 安装文档
date: 2019-10-16 21:13:06
tags:
categories:
	- Kubernetes
---


## 安装docker

卸载旧版本

```
# 在 master 节点和 worker 节点都要执行
yum remove -y docker \
docker-client \
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-selinux \
docker-engine-selinux \
docker-engine
```

设置 yum repository

```
# 在 master 节点和 worker 节点都要执行
yum install -y yum-utils \
device-mapper-persistent-data \
lvm2
sudo yum-config-manager \
--add-repo \
https://download.docker.com/linux/centos/docker-ce.repo
```

安装并启动 docker
```
# 在 master 节点和 worker 节点都要执行
yum install -y docker-ce-18.09.7 docker-ce-cli-18.09.7 containerd.io
systemctl enable docker
systemctl start docker
```

检查 docker 版本

```
# 在 master 节点和 worker 节点都要执行
docker version
```

### 安装可能遇到的报错
**Docker安装报错container-selinux >= 2.5-11**

这个报错是container-selinux版本低或者是没安装的原因，yum 安装container-selinux 一般的yum源又找不到这个包，需要安装epel源 才能yum安装container-selinux，然后再安装docker-ce就可以了

```
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum install epel-release   #阿里云上的epel源
yum install container-selinux
```

### 安装 nfs-utils

```
# 在 master 节点和 worker 节点都要执行
yum install -y nfs-utils
```

必须先安装 nfs-utils 才能挂载 nfs 网络存储

### K8S基本配置
配置K8S的yum源

```
# 在 master 节点和 worker 节点都要执行
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
```


关闭 SeLinux、swap
```
# 在 master 节点和 worker 节点都要执行


setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab
```


修改 /etc/sysctl.conf

# 在 master 节点和 worker 节点都要执行
vim /etc/sysctl.conf

向其中添加

```
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

如下所示

```
# System default settings live in /usr/lib/sysctl.d/00-system.conf.
# To override those settings, enter new settings here, or in an /etc/sysctl.d/<name>.conf file
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
kernel.core_uses_pid = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_timestamps = 0
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.all.arp_ignore = 1
vm.swappiness = 0
net.ipv4.ip_local_port_range = 22768 65000
kernel.shmmax = 68719476736
kernel.shmall = 4294967296

# Tcp add by SSG
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 0
net.core.netdev_max_backlog  = 262144
net.ipv4.tcp_max_syn_backlog = 262144
net.core.somaxconn = 65535
net.ipv4.tcp_max_orphans = 262144
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_max_tw_buckets = 40000
# Tcp end by SSG

net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

```

执行命令以应用

```
# 在 master 节点和 worker 节点都要执行
sysctl -p
```

安装kubelet、kubeadm、kubectl

```
# 在 master 节点和 worker 节点都要执行
yum install -y kubelet-1.15.1 kubeadm-1.15.1 kubectl-1.15.1
```

修改docker Cgroup Driver为systemd

```
# 在 master 节点和 worker 节点都要执行
vim /usr/lib/systemd/system/docker.service
```
向其中添加

```
--exec-opt native.cgroupdriver=systemd
```
```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
BindsTo=containerd.service
After=network-online.target firewalld.service containerd.service
Wants=network-online.target
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd
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

设置 docker 镜像

执行以下命令使用 docker 国内镜像，提高 docker 镜像下载速度和稳定性.如果您访问 https://hub.docker.io 速度非常稳定，亦可以跳过这个步骤

```
# 在 master 节点和 worker 节点都要执行
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
```

重启 docker，并启动 kubelet
```
# 在 master 节点和 worker 节点都要执行
systemctl daemon-reload
systemctl restart docker
systemctl enable kubelet && systemctl start kubelet
```

### 初始化 master 节点

以 root 身份在 demo-master-a-1 机器上执行。

创建 ./kubeadm-config.yaml

```
# 只在 master 节点执行
cat <<EOF > ./kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.15.1
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: "填上机器的ip或者域名:6443"
networking:
  podSubnet: "10.100.0.1/20"
EOF
```

podSubnet 所使用的网段不能与节点所在的网段重叠

初始化 apiserver
```
# 只在 master 节点执行
kubeadm init --config=kubeadm-config.yaml --upload-certs
```
根据您服务器网速的情况，您需要等候 1 – 10 分钟

初始化 root 用户的 kubectl 配置

```
# 只在 master 节点执行
rm -rf /root/.kube/
mkdir /root/.kube/
cp -i /etc/kubernetes/admin.conf /root/.kube/config
```

安装 calico

```
# 只在 master 节点执行
kubectl apply -f https://docs.projectcalico.org/v3.6/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
```

等待calico安装就绪：

执行如下命令，等待 3-10 分钟，直到所有的容器组处于 Running 状态
```
# 只在 master 节点执行
watch kubectl get pod -n kube-system
```
检查 master 初始化结果

在 master 节点 demo-master-a-1 上执行
```
# 只在 master 节点执行
kubectl get nodes
```


### 初始化 worker节点

获得 join命令参数

在 master 节点 demo-master-a-1 节点执行
```
# 只在 master 节点执行
kubeadm token create --print-join-command
```

可获取kubeadm join 命令及参数，如下所示

# kubeadm token create 命令的输出
```
kubeadm join apiserver.demo:6443 --token mpfjma.4vjjg8flqihor4vt     --discovery-token-ca-cert-hash sha256:6f7a8e40a810323672de5eee6f4d19aa2dbdb38411845a1bf5dd63485c43d303
```

#初始化worker
针对所有的 worker 节点执行(执行上面那个从master得到的指令)

```
# 只在 worker 节点执行
kubeadm join apiserver.demo:6443 --token mpfjma.4vjjg8flqihor4vt     --discovery-token-ca-cert-hash sha256:6f7a8e40a810323672de5eee6f4d19aa2dbdb38411845a1bf5dd63485c43d303
```
如果出现如下的报错
```
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR DirAvailable--etc-kubernetes-manifests]: /etc/kubernetes/manifests is not empty
	[ERROR FileAvailable--etc-kubernetes-kubelet.conf]: /etc/kubernetes/kubelet.conf already exists
	[ERROR Port-10250]: Port 10250 is in use
	[ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: /etc/kubernetes/pki/ca.crt already exists
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`

```

在节点上先执行如下命令，清理kubeadm的操作，然后再重新执行join 命令：
```
kubeadm reset
```

### 检查初始化结果

在 master 节点 demo-master-a-1 上执行
```
# 只在 master 节点执行
kubectl get nodes
```
```
NAME      STATUS   ROLES    AGE   VERSION
zhshx90   Ready    master   37m   v1.15.1
zhshx91   Ready    <none>   35    v1.15.1
```

### 移除 worker 节点

正常情况下，您无需移除 worker 节点，如果添加到集群出错，您可以移除 worker 节点，再重新尝试添加

在准备移除的 worker 节点上执行

```
# 只在 worker 节点执行
kubeadm reset
```
在 master 节点 demo-master-a-1 上执行

```
# 只在 master 节点执行
kubectl delete node demo-worker-x-x
```
将 demo-worker-x-x 替换为要移除的 worker 节点的名字
worker 节点的名字可以通过在节点 demo-master-a-1 上执行 kubectl get nodes 命令获得

### Kubernetes Dashboard的安装与坑


提前声明：因为dashboard是使用https访问的，所以有证书的问题，安装后无法访问请不要慌张。

获取yaml

Kubernetes 默认没有部署 Dashboard，可通过如下命令安装：

```
wget http://mirror.faasx.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

#### 证书的制作，如果不需要的可以跳过

生成证书通过openssl生成自签名证书即可，不再赘述，参考如下所示：



```
[root@master keys]# pwd
/root/keys
[root@master keys]# ls
[root@master keys]# openssl genrsa -out dashboard.key 2048
Generating RSA private key, 2048 bit long modulus
.+++
.................................................+++
e is 65537 (0x10001)
[root@master keys]# openssl req -new -out dashboard.csr -key dashboard.key -subj '/CN=本机的ip'
[root@master keys]# ls
dashboard.csr  dashboard.key
[root@master keys]# 
[root@master keys]# openssl x509 -req -in dashboard.csr -signkey dashboard.key -out dashboard.crt 
Signature ok
subject=/CN=192.168.246.200
Getting Private key
[root@master keys]# 
[root@master keys]# ls
dashboard.crt  dashboard.csr  dashboard.key
[root@master keys]# 
[root@master keys]# openssl x509 -in dashboard.crt -text -noout
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number:
            f0:8a:26:aa:9f:24:bf:92
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=192.168.246.200
        Validity
            Not Before: Dec 13 08:10:36 2018 GMT
            Not After : Jan 12 08:10:36 2019 GMT
        Subject: CN=192.168.246.200
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:f6:7a:b4:4a:ad:bd:b3:00:9c:d1:fe:06:2d:09:
                    cf:eb:28:54:0f:3f:6e:dc:29:6b:67:e1:9b:58:e4:
                    82:00:15:ee:35:25:00:4c:c1:e0:1b:29:8b:b2:6b:
                    8d:e8:09:77:66:4d:f3:9e:9d:85:36:94:80:da:1b:
                    35:c8:a1:b3:0b:b2:7f:6f:1e:e9:fe:fc:15:1b:7b:
                    ba:85:1f:2b:70:16:d5:c3:7f:36:18:f1:8e:44:1e:
                    8a:13:a2:9c:b8:bf:b8:08:3f:a0:5c:ef:19:f5:ce:
                    73:0c:3e:0a:b5:87:7a:de:25:74:36:0e:26:52:ff:
                    4b:d0:24:40:c9:03:9a:44:f6:17:a7:d7:fa:7e:e0:
                    fb:6a:76:5b:dc:0f:43:c2:63:f4:22:20:4c:4e:5d:
                    b7:a0:83:54:58:1c:10:0f:57:ef:ad:1f:36:0b:8f:
                    8d:f4:a2:52:ab:e7:39:57:ea:30:c3:1d:30:93:ee:
                    44:7f:73:ef:41:94:e8:34:8c:c4:bb:02:d9:17:da:
                    55:07:ff:43:6c:f3:8e:91:5f:81:03:e9:94:2e:f1:
                    25:e7:41:86:e2:25:c4:b9:07:b4:9c:d9:04:36:31:
                    82:43:1b:26:10:17:8c:98:4a:f3:23:69:15:1b:76:
                    75:ae:4e:27:6f:70:4c:c6:f7:cc:75:e4:ed:48:b7:
                    51:c5
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha256WithRSAEncryption
         28:55:3c:0a:66:77:2a:fd:8a:b6:81:54:59:13:d7:03:17:7f:
         d4:fa:e4:94:2b:bc:f4:11:ea:0c:18:e9:c0:2c:02:86:eb:39:
         12:38:19:71:6c:b8:7a:4d:03:57:59:4f:c0:50:c4:19:92:c1:
         9f:2f:0d:18:92:9e:2b:2e:a2:44:52:9a:32:2b:75:35:fb:43:
         66:fb:fa:32:77:ce:b8:4e:80:cb:38:52:c4:2c:17:11:1a:38:
         c3:a9:62:43:5e:60:ae:47:d4:f7:46:12:29:f5:e4:75:35:e5:
         90:5d:2e:4f:2f:c5:65:9a:e5:6a:4d:8a:cd:69:ba:e0:4f:43:
         d1:ab:9a:62:74:fc:d5:88:9c:3a:ba:22:2d:38:96:fc:35:b0:
         3c:23:f7:8c:23:07:4e:05:8e:ae:53:82:9c:fd:54:24:86:75:
         12:a6:e9:77:62:bd:f6:bb:f9:4d:5b:64:1e:d0:48:68:31:86:
         f5:36:b5:6b:fc:b6:36:f0:01:3c:0a:9f:2b:27:56:28:1d:1f:
         c4:e9:f7:c6:5d:16:5e:88:c5:e0:43:00:bf:79:d7:04:2f:45:
         57:df:e6:17:dd:5a:f8:53:e9:ca:f6:33:ed:19:f0:d9:0a:ae:
         f0:ba:c6:5b:7e:70:af:c3:f3:a5:b0:95:b0:ee:cd:35:29:5c:
         34:4a:ce:49
```

这样就有了证书文件dashboard.crt 和 私钥 dashboad.key

将该配置文件中创建secret的配置文件信息去掉，将以下内容 从配置文件中去掉：

```
# ------------------- Dashboard Secret ------------------- #

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque

---
```
可以在配置文件中，修改service 为nodeport类型，固定访问端口
修改前：


```
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:

  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
```

修改后： 修改两个地方。type: NodePort 和  nodePort:30001
 
```
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      nodePort:30001
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
```

生成secret

创建同名称的secret：
名称为： kubernetes-dashboard-certs

```
[root@master keys]# ls
dashboard.crt  dashboard.csr  dashboard.key  kubernetes-dashboard.yaml
[root@master keys]# kubectl -n kube-system  create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt 
secret/kubernetes-dashboard-certs created
[root@master keys]# 
[root@master keys]# kubectl -n kube-system  get secret | grep dashboard
kubernetes-dashboard-certs                        Opaque                                2      25s
kubernetes-dashboard-key-holder                  Opaque                                2      25h
[root@master keys]# 
[root@master keys]# kubectl -n kube-system  describe secret kubernetes-dashboard-certs  
Name:         kubernetes-dashboard-certs
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
dashboard.crt:  993 bytes
dashboard.key:  1675 bytes
[root@master keys]# 
```

部署apply yaml文件

```
kubectl apply -f kubernetes-dashboard.yaml 
```

查看服务状态：

```
[root@master keys]# kubectl -n kube-system get svc
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP   15d
kubernetes-dashboard   NodePort    10.111.32.20   <none>        443:30001/TCP   2m14s
[root@master keys]# 
```


![image](https://note.youdao.com/yws/api/personal/file/C2982ED8E7EA4E3283950AFDEE8EF174?method=download&shareKey=7b43d29eb9f1fd10f747baf5fd5ed74b)

不过有些同学先装kubernetes-dashboard，得先删除对应的service和证书，需要通过下面的指令
```
kubectl delete -f kubernetes-dashboard.yml 
```
如果是通过下面的方式直接装的删除语句就得修改了。

```
kubectl apply -f http://mirror.faasx.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

```
kubectl delete -f  http://mirror.faasx.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

#### 登录
Dashboard 支持 Kubeconfig 和 Token 两种认证方式，为了简化配置，我们通过配置文件 dashboard-admin.yaml 为 Dashboard 默认用户赋予 admin 权限。

```
[root@ken ~]# cat dashboard-admin.yml

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels: 
     k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
 ```
 
 执行kubectl apply使之生效
 ```
 kubectl apply -f dashboard-admin.yml
 ```
 现在直接点击登录页面的 SKIP 就可以进入 Dashboard 了。
 
 
 token模式
 
 获取token 这里有一个简单的命令：
```
kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token

```
### 坑
ui可能会乱跑。。不一定在生成的证书的那台服务器

需要修改yml，配置nodeSelector来指定pod落在指定的node上。


![image](https://note.youdao.com/yws/api/personal/file/D98E066CD35D47DA9F4778C9F57849DB?method=download&shareKey=c8a2b183a96d6262b48340b5beb2996d)


### Reference


https://www.kubernetes.org.cn/5650.html     
https://www.jianshu.com/p/c6d560d12d50     
https://www.cnblogs.com/kenken2018/p/10340157.html    
https://blog.csdn.net/supermao1013/article/details/85050781    