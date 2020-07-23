---
title: 一次kubernetes网络问题排查
date: 2020-06-28 22:31:49
tags:
categories:
	- Kubernetes
---

## 前言


笔者所用个的kubernetes版本为1.17，对应的cni使用的是weave。其实想用calico或者flannel gw，性能会更好，奈何需要自己组建内网环境，这明显就是超纲了。





## 正文



大约是在早上10点的时候，整个集群网络互相不能通信，容器内解析域名全部失败 UnknownHostException，在宿主机上ping baidu.com是正常的。排除掉宿主机的网络问题，初步怀疑是cni出问题了，导致dns解析有问题，容器内解析域名出错。



```shell
sudo curl -L git.io/weave -o /usr/local/bin/weave

sudo chmod a+x /usr/local/bin/weave
 
```



在机器上通过如上方式安装好weave指令后，可以通过如下指令看到对应的连接情况。



```shell
weave --local status
weave --local status connections
weave --local status ipam
weave --local report
weave --local status peers
```





通过weave --local status ，看到52个连接中有2个连接是失败的。



```shell

        Version: 2.6.2 (failed to check latest version - see logs; next check at 2020/06/28 07:01:40)

        Service: router
       Protocol: weave 1..2
           Name: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
     Encryption: disabled
  PeerDiscovery: enabled
        Targets: 52
    Connections: 52 (50 established,2 failed)
          Peers: 53 (with 2756 established connections)
 TrustedSubnets: none

        Service: ipam
         Status: ready
          Range: 10.32.0.0/12
  DefaultSubnet: 10.32.0.0/12

```



weave --local status connections查看具体情况



```shell
-> xxxxx:6378    failed Multiple connections to xxxxxxxxxxx 
-> xxxxx:6378    failed Multiple connections to xxxxxxxxxxx 
```



有两台机器一直都是报错状态，到多台机器上执行，获得了一坨的ip，这些ip很神奇的都是双线ip，上面既有电信ip又有联通ip，怀疑是底层路由表有问题，weave用电信ip去连接另一台机器的联调ip，随即下掉一台机器的联通ip，出现了io timeout的报错，证实了用联通ip去连接电信ip的猜想。



```shell
->  xxxx:6783  retrying red tcp xxx:xx -> xxxx:xx  i/o timeout
```

  

下掉机器的联通ip后，重启weave，weave --local status看到的全是正常连接了。





### 总结



网络是一个天坑，基础建设很重要，后续全部走内网ip吧，如果内网跨交换机还出问题，那只能打网络组了。