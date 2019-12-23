---
title: Flink on Kubernetes 搭建
date: 2019-12-23 21:42:04
tags:
categories:
	- Flink
---

依赖镜像制作

```
FROM  flink:1.9.1-scala_2.12
ADD ./flink-shaded-hadoop-2-uber-2.8.3-7.0.jar /opt/flink/lib/
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone

```

制作镜像
```
docker build -t='study/flink:1.9-2.8' .
docker tag study/flink:1.9-2.8 k8stest.net:8876/study/flink
```

提交镜像到仓库
```
docker push k8stest.net:8876/study/flink
```

密码
```
kubectl create secret docker-registry harborregsecret --docker-server=k8stest.net:8876 --docker-username=admin --docker-password=123456 -n flink 
```


installFlink.sh
```
namespace=$1
if [ !  $namespace ]; then  
 echo "error : nameSpace is NULL";
 exit 0
fi 
echo 'nameSpace is '$namespace
kubectl apply -f flink-configuration-configmap.yaml  -n $namespace
kubectl apply -f jobmanager-service.yaml -n $namespace
kubectl apply -f jobmanager-rest-service.yaml -n $namespace
kubectl apply -f jobmanager-deployment.yaml -n $namespace
kubectl apply -f taskmanager-deployment.yaml -n $namespace

```

deleteFlink.sh
```
namespace=$1
if [ !  $namespace ]; then  
 echo "error : nameSpace is NULL";
 exit 0
fi 
echo 'nameSpace is '$namespace
kubectl delete -f flink-configuration-configmap.yaml  -n $namespace
kubectl delete -f jobmanager-service.yaml -n $namespace
kubectl delete -f jobmanager-deployment.yaml -n $namespace
kubectl delete -f jobmanager-rest-service.yaml -n $namespace
kubectl delete -f taskmanager-deployment.yaml -n $namespace

```

flink-configuration-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: flink-config
  labels:
    app: flink
data:
  flink-conf.yaml: |+
    jobmanager.rpc.address: flink-jobmanager
    taskmanager.numberOfTaskSlots: 1
    blob.server.port: 6124
    jobmanager.rpc.port: 6123
    taskmanager.rpc.port: 6122
    jobmanager.heap.size: 1024m
    taskmanager.heap.size: 1024m
  log4j.properties: |+
    log4j.rootLogger=INFO, file
    log4j.logger.akka=INFO
    log4j.logger.org.apache.kafka=INFO
    log4j.logger.org.apache.hadoop=INFO
    log4j.logger.org.apache.zookeeper=INFO
    log4j.appender.file=org.apache.log4j.FileAppender
    log4j.appender.file.file=${log.file}
    log4j.appender.file.layout=org.apache.log4j.PatternLayout
    log4j.appender.file.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %-5p %-60c %x - %m%n
    log4j.logger.org.apache.flink.shaded.akka.org.jboss.netty.channel.DefaultChannelPipeline=ERROR, file

```


jobmanager-deployment.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: flink-jobmanager
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: flink
        component: jobmanager
    spec:
      containers:
      - name: jobmanager
        image: flink:latest
        workingDir: /opt/flink
        command: ["/bin/bash", "-c", "$FLINK_HOME/bin/jobmanager.sh start;\
          while :;
          do
            if [[ -f $(find log -name '*jobmanager*.log' -print -quit) ]];
              then tail -f -n +1 log/*jobmanager*.log;
            fi;
          done"]
        ports:
        - containerPort: 6123
          name: rpc
        - containerPort: 6124
          name: blob
        - containerPort: 8081
          name: ui
        livenessProbe:
          tcpSocket:
            port: 6123
          initialDelaySeconds: 30
          periodSeconds: 60
        volumeMounts:
        - name: flink-config-volume
          mountPath: /opt/flink/conf
      volumes:
      - name: flink-config-volume
        configMap:
          name: flink-config
          items:
          - key: flink-conf.yaml
            path: flink-conf.yaml
          - key: log4j.properties
            path: log4j.properties

```

jobmanager-rest-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flink-jobmanager-rest
spec:
  type: NodePort
  ports:
  - name: rest
    port: 8081
    targetPort: 8081
  selector:
    app: flink
    component: jobmanager

```

 jobmanager-service.yaml
 
```yaml
apiVersion: v1
kind: Service
metadata:
  name: flink-jobmanager
spec:
  type: ClusterIP
  ports:
  - name: rpc
    port: 6123
  - name: blob
    port: 6124
  - name: ui
    port: 8081
  selector:
    app: flink
    component: jobmanager

```

taskmanager-deployment.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: flink-taskmanager
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: flink
        component: taskmanager
    spec:
      containers:
      - name: taskmanager
        image: flink:latest
        workingDir: /opt/flink
        command: ["/bin/bash", "-c", "$FLINK_HOME/bin/taskmanager.sh start; \
          while :;
          do
            if [[ -f $(find log -name '*taskmanager*.log' -print -quit) ]];
              then tail -f -n +1 log/*taskmanager*.log;
            fi;
          done"]
        ports:
        - containerPort: 6122
          name: rpc
        livenessProbe:
          tcpSocket:
            port: 6122
          initialDelaySeconds: 30
          periodSeconds: 60
        volumeMounts:
        - name: flink-config-volume
          mountPath: /opt/flink/conf/
      volumes:
      - name: flink-config-volume
        configMap:
          name: flink-config
          items:
          - key: flink-conf.yaml
            path: flink-conf.yaml
          - key: log4j.properties
            path: log4j.properties
```



### 固定jm在具体机器


如果jobmanger随便飘，提交会有问题。

包含nodeSelector的jobmanager-deployment.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: flink-jobmanager
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: flink
        component: jobmanager
    spec:
      nodeSelector:
        flink: jm
      containers:
      - name: jobmanager
        image: flink:latest
        workingDir: /opt/flink
        command: ["/bin/bash", "-c", "$FLINK_HOME/bin/jobmanager.sh start;\
          while :;
          do
            if [[ -f $(find log -name '*jobmanager*.log' -print -quit) ]];
              then tail -f -n +1 log/*jobmanager*.log;
            fi;
          done"]
        ports:
        - containerPort: 6123
          name: rpc
        - containerPort: 6124
          name: blob
        - containerPort: 8081
          name: ui
        livenessProbe:
          tcpSocket:
            port: 6123
          initialDelaySeconds: 30
          periodSeconds: 60
        volumeMounts:
        - name: flink-config-volume
          mountPath: /opt/flink/conf
      volumes:
      - name: flink-config-volume
        configMap:
          name: flink-config
          items:
          - key: flink-conf.yaml
            path: flink-conf.yaml
          - key: log4j.properties
            path: log4j.properties

```

### 固定flink ui的端口

jobmanager-rest-service.yaml

通过增加nodePort: 30000来固定flink端口


```yaml
apiVersion: v1
kind: Service
metadata:
  name: flink-jobmanager-rest
spec:
  type: NodePort
  ports:
  - name: rest
    port: 8081
    targetPort: 8081
    nodePort: 30000
  selector:
    app: flink
    component: jobmanager

```



### HA版本的搭建

在配置文件中添加如下信息

```
high-availability: zookeeper
high-availability.storageDir: hdfs://hdfs-address/flink/recovery
high-availability.zookeeper.quorum: zk-address:7072
high-availability.zookeeper.path.root: /flink
high-availability.jobmanager.port: 6123
high-availability.cluster-id: xxxx（自取）
```

特别注意，ha的jobmanager.port选择jm的rpc端口。


### 状态后台配置

```
state.checkpoints.dir: hdfs://hdfs-address/flink/flink-checkpoints
state.savepoints.dir: hdfs://hdfs-address/flink/flink-savepoints
state.backend: rocksdb
```

### Reference 



https://ci.apache.org/projects/flink/flink-docs-release-1.9/ops/deployment/kubernetes.html


HA:
http://shzhangji.com/cnblogs/2019/08/25/deploy-flink-job-cluster-on-kubernetes/
