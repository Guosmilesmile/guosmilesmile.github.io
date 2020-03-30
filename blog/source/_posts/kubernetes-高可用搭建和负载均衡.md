---
title: kubernetes 高可用搭建和负载均衡
date: 2020-02-01 22:18:25
tags:
categories:
	- Kubernetes

---


### 架构图
![image](https://note.youdao.com/yws/api/personal/file/77FAA4B774334108A5DB32F4270EB0FE?method=download&shareKey=0ffc47993e10884612f3ce3ffb9fec00)

![image](https://note.youdao.com/yws/api/personal/file/6EE58799617148DB93B562D02D8DE049?method=download&shareKey=b0e00f8bc194046c595b4b385ff76b39)

### 原理

安装keepalive，vip对外，没有ng。那么其实对外提供服务的master只有一台，除非这台挂了才会切换到下一台 。


只用keepalived实现master ha，当api-server的访问量大的时候，会有性能瓶颈问题，通过配置haproxy，可以同时实现master的ha和流量的负载均衡。

可是，如果有一台master挂了，vip会飘到新的机器，高可用了，可是ng上的配置，还是三台呀，还是会转发到挂的机器 

加了ng岂不是更没办法做到高可用 ？


除非ng或者haproxy可以做到，对后端的服务做探活，如果挂了就T掉服务。

haproxy提供了现成的功能，如果使用ng的话，可以配合openresty实现

```
backend https_sri
    balance      roundrobin
    server s1 192.168.115.5:6443  check inter 10000 fall 2 rise 2 weight 1
    server s2 192.168.115.6:6443  check inter 10000 fall 2 rise 2 weight 1
```

* check：表示启用对此后端服务器执行健康状态检查。
* inter：设置健康状态检查的时间间隔，单位为毫秒。
* rise：设置从故障状态转换至正常状态需要成功检查的次数，例如。“rise 2”表示2 次检查正确就认为此服务器可用。
* fall：设置后端服务器从正常状态转换为不可用状态需要检查的次数，例如，“fall 3”表示3 次检查失败就认为此服务器不可用。


### 注意

三个 master 组成主节点集群，通过内网 loader balancer 实现负载均衡；至少需要三个 master 节点才可组成高可用集群，否则会出现 脑裂 现象。 

最多挂一台，如果挂两台，集群就失效了。


### 安装和配置keepalived

```
yum -y install keepalived
```

主的配置
```

global_defs {
    smtp_server 127.0.0.1
    smtp_connect_timeout 30
    router_id LVS
}

vrrp_script check_kube {
    script "/opt/keepalived-check/kube.sh"
    interval 3
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface bond0
    virtual_router_id 60
    priority 110
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass admin123456
    }
    virtual_ipaddress {
       10.17.134.5
    }

    track_script {
        check_kube
    }
}
```


备的配置

```
global_defs {
    smtp_server 127.0.0.1  
    smtp_connect_timeout 30  
    router_id LVS
}

vrrp_script check_kube {
    script "/opt/keepalived-check/kube.sh"
    interval 3
    weight 2
}

vrrp_instance VI_1 {  
    state BACKUP
    interface eth4
    virtual_router_id 60  
    priority 109  
    advert_int 1  
    authentication {  
        auth_type PASS  
        auth_pass admin123456
    }  
    virtual_ipaddress {  
       10.17.134.5
    }

    track_script {   
        check_kube
    }
}



```

检测脚本

/opt/keepalived-check/kube.sh

```
#!/bin/bash

/usr/bin/killall -0 kube-apiserver 2>/dev/null && exit 0 || exit 1

```

如果前面是架haproxy，那么用如下脚本。因为keepalive探测haproxy的死活，haproxy探测kube-apiserver的死活。

/opt/keepalived-check/haproxy.sh

```
#!/bin/bash

/usr/bin/killall -0 haproxy 2>/dev/null && exit 0 || exit 1

```

如果是bak节点，需要修改state为BACKUP, priority为99 （priority值必须小于master节点配置值）

### haproxy的安装

只用keepalived实现master ha，当api-server的访问量大的时候，会有性能瓶颈问题，通过配置haproxy，可以同时实现master的ha和流量的负载均衡。


```
yum -y install haproxy
```


配置文件
/etc/haproxy/haproxy.cfg 

```
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    #log         127.0.0.1 local2
    log 127.0.0.1 local0 info
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
     # turn on stats unix socket
    stats socket /var/lib/haproxy/stats


    defaults
        mode                    tcp
        log                     global
        retries                 3
        timeout connect         10s
        timeout client          1m
        timeout server          1m
    
    frontend  kubernetes
            bind *:8443
            mode tcp
            default_backend kubernetes_master
    
    backend kubernetes_master
        balance     roundrobin
        server master master_ip:6443 check maxconn 2000
        server master1 master1_ip:6443 check maxconn 2000
        server master2 master2_ip:6443 check maxconn 2000

```

### haproxy开启日志功能

安装部署完Haproxy之后，默认是没有开启日志记录的，需要相应的手工配置使其日志功能开启。

* 【创建日志记录文件夹】


```
mkdir /var/log/haproxy
chmod a+x /var/log/haproxy
```

*【开启rsyslog记录haproxy日志功能】

vim /etc/rsyslog.conf

修改：
```
# Provides UDP syslog reception

$ModLoad imudp

$UDPServerRun 514
```
开启514 UDP监听

添加：
```
# Save haproxy log

local0.* /var/log/haproxy/haproxy.log
```

* 【修改/etc/sysconfig/rsyslog】
vim /etc/sysconfig/rsyslog

```
# Options for rsyslogd

# Syslogd options are deprecated since rsyslog v3.

# If you want to use them, switch to compatibility mode 2 by "-c 2"

# See rsyslogd(8) for more details

SYSLOGD_OPTIONS="-r -m 0 -c 2"
```

* 【haproxy配置】

vim /etc/haproxy/haproxy.conf

添加：
```
global #在此上级目录下配置

log 127.0.0.1 local0 info
```
配置local0事件

* 【验证服务是否生效】

```
###重启服务

systemctl restart haproxy

systemctl restart rsyslog

   

###查看日志记录

tailf /var/log/haproxy/haproxy.log
```


### 新master的添加
高可用的安装方式就存在不一样的地方了。


####  一、 kubeadm init的配置文件

kubeadm-config.yaml
```
# 只在 master 节点执行
cat <<EOF > ./kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.15.1
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: "填上机器的ip或者域名:8443(haproxy监听的端口)"
networking:
  podSubnet: "10.100.0.1/20"
EOF

```

其他的可以参考这篇文章
[Kubernetes 、Docker 和 Dashboard 安装文档](https://guosmilesmile.github.io/2019/10/16/Kubernetes-%E3%80%81Docker-%E5%92%8C-Dashboard-%E5%AE%89%E8%A3%85%E6%96%87%E6%A1%A3/)

一、首先在master上生成新的token

```
kubeadm token create --print-join-command
 [root@cn-hongkong nfs]# kubeadm token create --print-join-command
kubeadm join 172.31.182.156:8443 --token ortvag.ra0654faci8y8903     --discovery-token-ca-cert-hash sha256:04755ff1aa88e7db283c85589bee31fabb7d32186612778e53a536a297fc9010
```
二、在master上生成用于新master加入的证书

```
kubeadm init phase upload-certs --upload-certs
[root@cn-hongkong k8s_yaml]# kubeadm init phase upload-certs --experimental-upload-certs
[upload-certs] Storing the certificates in ConfigMap "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
f8d1c027c01baef6985ddf24266641b7c64f9fd922b15a32fce40b6b4b21e47d
```

 三、添加新master，把红色部分加到--experimental-control-plane --certificate-key后。
```

kubeadm join 172.31.182.156:8443  --token ortvag.ra0654faci8y8903 \
  --discovery-token-ca-cert-hash sha256:04755ff1aa88e7db283c85589bee31fabb7d32186612778e53a536a297fc9010 \
  --control-plane --certificate-key f8d1c027c01baef6985ddf24266641b7c64f9fd922b15a32fce40b6b4b21e47d
 ```

四、初始化 root 用户的 kubectl 配置

每台master都要操作
```
rm -rf /root/.kube/
mkdir /root/.kube/
cp -i /etc/kubernetes/admin.conf /root/.kube/config
```


### 总结

kubernetes的高可用，是通过访问vip：haproxy监听端口，vip会飘逸到可用的服务器上。然后通过haproxy转发到真实的端口，可能是本机也可能是其他master。haproxy本身具有探活功能。整个高可用就串起来了。
