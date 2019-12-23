---
title: harbor安装
date: 2019-12-23 21:16:28
tags:
categories:
	- Kubernetes
---


### 下载地址

官网下载地址
```
https://goharbor.io/
```
或者下面的地址
```
https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-offline-installer-v1.8.0.tgz
```

### 安装

解压压缩包

```
tar   -zxvf   harbor-offline-installer-v1.8.0.tgz 
```

#### 配置harbor.yml  


较重要参数说明

*   hostname  目标主机的主机名，用于访问Portal和注册表服务。它应该是目标计算机的IP地址或完全限定的域名（FQDN），例如，192.168.1.10或reg.yourdomain.com。不要使用localhost或127.0.0.1作为主机名 - 外部客户端需要访问注册表服务
```
这里修改为我们的主机ip即可  例如修改为  10.10.55.55
```
* data_volume： 存储  harbor  数据的位置。   这里可以修改 为   /usr/local/workspace/harbor/data

* arbor_admin_password：管理员的初始密码。此密码仅在Harbor首次启动时生效。之后，将忽略此设置，并且应在Portal中设置管理员密码。请注意，默认用户名/密码为admin / Harbor12345。

关于端口配置：

http：
port：你的http的端口号
https：用于访问Portal和令牌/通知服务的协议。如果启用了公证，则必须设置为https。请参阅使用HTTPS访问配置Harbor。

port：https的端口号
certificate：SSL证书的路径，仅在协议设置为https时应用。
private_key：SSL密钥的路径，仅在协议设置为https时应用。


### 执行  ./install.sh

```
执行  ./prepare
./prepare


# 执行 ./install.sh
./install.sh

# 查看启动情况
docker-compose ps
```
### docker-compose安装

安装pip
```
yum -y install epel-release
yum -y install python-pip
```
确认版本
```
pip --version
```
更新pip
```
pip install --upgrade pip
```
安装docker-compose
```
pip install docker-compose 
```
or 

```
pip install docker-compose   --ignore-installed requests
```
查看版本
```
docker-compose version
```



### 4.使用


4.1  配置免https
方法一：修改  /etc/docker/daemon.json

```
vi /etc/docker/daemon.json
```


 加上 允许的仓库
```
{
  "insecure-registries":["http://k8stest.net:8876"]
}
```

### 重启docker 

```
systemctl daemon-reload
systemctl restart docker.service

# 重启harbor仓库
# cd 到 harbor的安装目录

# 执行命令
docker-compose stop
docker-compose up -d
```

如果出现k8s一起出问题，需要重启k8s


设置密码
```
kubectl create secret docker-registry harbor-regsecret --docker-server=kube-master.net:8876 --docker-username=admin --docker-password=admin123456 -n flink 
```



### Reference

https://blog.csdn.net/liyin6847/article/details/90600270

https://www.cnblogs.com/haoprogrammer/p/11023926.html

