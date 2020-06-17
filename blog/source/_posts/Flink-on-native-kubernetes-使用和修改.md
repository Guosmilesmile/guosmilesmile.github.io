---
title: Flink on native kubernetes 使用和修改
date: 2020-05-27 21:00:26
tags:
categories:
	- Flink
---



### 背景

目前flink on kubernetes的版本是standalone，资源释放的问题是一个比较头大的问题，如果作业cancel，程序开了别的线程或者内存出现泄漏，都会导致TM有问题。

native kubernetes的seesion模式可以比较好的解决，跟yarn模式一样，可以较好的解决该问题。


### 版本
组件 | 版本
---|---
flink | 1.10.1
kubernetes | 1.17.4
java  | jdk-8u252

以上版本比较麻烦，会出现特殊情况，需要自行构建flink镜像，降低jdk8的版本


组件 | 版本
---|---
flink | 1.10.1
kubernetes | 1.15.1
java  | jdk-8u252

以上版本可以直接根据官方方法建构

### Native Kubernetes Setup


#### 创建好角色和赋权

rbac.yaml 

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flink
  namespace: flink
---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: flink-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
- kind: ServiceAccount
  name: flink
  namespace: flink
```


##### 修改日志输出

默认情况下, JobManager 和 TaskManager 只会将 log 写到各自 pod 的 /opt/flink/log 。如果想通过 kubectl logs 看到日志，需要将 log 输出到控制台。要做如下修改 FLINK_HOME/conf 目录下的 log4j.properties 文件。(修改提交机即可)

log4j.rootLogger=INFO, file, console
 
```
# Log all infos to the console
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %-5p %-60c %x - %m%n
```

然后启动 session cluster 的命令行需要带上参数:


```
-Dkubernetes.container-start-command-template="%java% %classpath% %jvmmem% %jvmopts% %logging% %class% %args%"
```


#### 启动 session cluster

如下命令是启动一个每个 TaskManager 是4G内存，2个CPU，4个slot 的 session cluster。

```
bin/kubernetes-session.sh -Dkubernetes.container-start-command-template="%java% %classpath% %jvmmem% %jvmopts% %logging% %class% %args%" -Dkubernetes.cluster-id=kaibo-test -Dtaskmanager.memory.process.size=4096m -Dkubernetes.taskmanager.cpu=2 -Dtaskmanager.numberOfTaskSlots=4

```

如果其他集群参数需要添加的，通过-D继续补充

```
/usr/local/flink/flink-1.10.1/bin/kubernetes-session.sh \
          -Dkubernetes.cluster-id=ipcode \
          -Dkubernetes.jobmanager.service-account=flink \
          -Dtaskmanager.memory.process.size=4096m \
          -Dkubernetes.taskmanager.cpu=2 \
          -Dtaskmanager.numberOfTaskSlots=1 \
          -Dkubernetes.namespace=flink-ipcode \
          -Djobstore.expiration-time=172800 \
          -Dtaskmanager.memory.managed.fraction=0.2 \
          -Dtaskmanager.memory.jvm-metaspace.size=256m \
          -Dkubernetes.container-start-command-template="%java% %classpath% %jvmmem% %jvmopts% %logging% %class% %args%" \
          -Dakka.framesize=104857600b \
          -Dkubernetes.container.image.pull-secrets=harbor-regsecret \
          -Dkubernetes.container.image=kube-master.net:8876/flink:1.10.1-8u242-1
```

#### 提交任务

```
/usr/local/flink/flink-1.10.1/bin/flink run -d  -e kubernetes-session -Dkubernetes.cluster-id=ipcode  -Dkubernetes.namespace=flink-ipcode  -c study.IpCodeToHdfsFlink project-1.0.0.jar

```

指定命名空间等参数，才可以提交到对应的集群。

#### 取消集群

```
echo 'stop' | /usr/local/flink/flink-1.10.1/bin/kubernetes-session.sh -Dkubernetes.cluster-id=ipcode  -Dkubernetes.namespace=flink-ipcode  -Dexecution.attached=true
```


### flink 1.10.1 和kubernetes 1.17.4的冲突

镜像的jdk版本是java 8u252，目前Flink on K8s不能和java 8u252一起工作，
解法是使用8u252以下的jdk版本或者升级到jdk11


http://apache-flink.147419.n8.nabble.com/native-kubernetes-kubernetes-td3360.html



### flink 降低jdk版本，构建自己的镜像


从https://github.com/apache/flink-docker/tree/master/1.10/scala_2.11-debian
上获取Dockerfile和docker-entrypoint.sh，放于同级目录。

将jdk改为

```
FROM adoptopenjdk/openjdk8:jdk8u242-b08
```

apt-get多下载wget和gnupg，可以多下载需要的工具，例如vim等等

```
# Install dependencies
RUN set -ex; \
  apt-get update; \
  apt-get -y install libsnappy1v5 gettext-base vim wget gnupg; \
  rm -rf /var/lib/apt/lists/*

```

修改镜像的时区

```
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone
```
添加需要的依赖到镜像中

```
ADD ./lib/* /opt/flink/lib/
```

```
[root@cz-flink-live-master flink-1.10.1]# ll
total 20
-rw-r--r-- 1 root root 4145 May 27 14:54 docker-entrypoint.sh
-rw-r--r-- 1 root root 3751 May 27 18:58 Dockerfile
drwxr-xr-x 2 root root 4096 May 27 18:45 lib
-rw-r--r-- 1 root root  151 May 27 18:53 push.sh
[root@cz-flink-live-master flink-1.10.1]# ls lib/
flink-metric-1.0.3.jar  flink-metrics-core-1.10.1.jar  flink-metrics-prometheus_2.11-1.10.1.jar  flink-shaded-hadoop-2-uber-2.8.3-7.0.jar

```

最终文件如下

```
FROM adoptopenjdk/openjdk8:jdk8u242-b08

# Install dependencies
RUN set -ex; \
  apt-get update; \
  apt-get -y install libsnappy1v5 gettext-base vim wget gnupg; \
  rm -rf /var/lib/apt/lists/*

# Grab gosu for easy step-down from root
ENV GOSU_VERSION 1.11
RUN set -ex; \
  wget -nv -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)"; \
  wget -nv -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc"; \
  export GNUPGHOME="$(mktemp -d)"; \
  for server in ha.pool.sks-keyservers.net $(shuf -e \
                          hkp://p80.pool.sks-keyservers.net:80 \
                          keyserver.ubuntu.com \
                          hkp://keyserver.ubuntu.com:80 \
                          pgp.mit.edu) ; do \
      gpg --batch --keyserver "$server" --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 && break || : ; \
  done && \
  gpg --batch --verify flink.tgz.asc flink.tgz; \
    gpgconf --kill all; \
    rm -rf "$GNUPGHOME" flink.tgz.asc; \
  fi; \
  \
  tar -xf flink.tgz --strip-components=1; \
  rm flink.tgz; \
  \
  chown -R flink:flink .;
# Configure container
COPY docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
EXPOSE 6123 8081
CMD ["help"]
ADD ./lib/* /opt/flink/lib/
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone
```




通过如下指令构建镜像，推到私人仓库上

```
docker build -t='kube-master.net:8876/study/flink:1.10.1-8u242-1' .
docker push kube-master.net:8876/study/flink:1.10.1-8u242-1

```


### 最新解决方法


看到git上关于kubernetes-client could not work with java 8u252[1]的问题。根据flink英文邮件列表[2]中的方法添加如下参数，可以正常解决jdk版本的问题
-Dcontainerized.master.env.HTTP2_DISABLE=true


[1] https://github.com/fabric8io/kubernetes-client/issues/2212       
[2] http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/Native-K8S-not-creating-TMs-td35703.html



### Attention 

现在Flink 1.10.1版本，没办法指定私有仓库的secret，需要等1.11版本才支持，第一次需要在每台服务器上提前拉去镜像，这点比较麻烦。

```
-Dkubernetes.container.image.pull-secrets=harbor-regsecret 
```











### Reference

[flink-docker git官方镜像地址](https://github.com/apache/flink-docker)

[Native Kubernetes Setup Beta](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/deployment/native_kubernetes.html)


[native kubernetes在不同kubernetes版本构建失败问题](http://apache-flink.147419.n8.nabble.com/native-kubernetes-kubernetes-td3360.html)