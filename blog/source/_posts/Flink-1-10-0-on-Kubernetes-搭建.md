---
title: Flink 1.10.0 on Kubernetes 搭建
date: 2020-03-30 16:32:05
tags:
categories:
	- Flink
---
依赖镜像制作

```
FROM  flink:1.10.0-scala_2.11
ADD ./lib/* /opt/flink/lib/
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone
```

制作镜像
```
docker build -t='k8stest.net:8876/study/flink:1.10.0' .

```

提交镜像到仓库
```
docker push k8stest.net:8876/study/flink:1.10.0
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
    taskmanager.memory.process.size: 1024m
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
        image: k8stest.net:8876/study/flink:1.10.0
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
        image: k8stest.net:8876/study/flink:1.10.0
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
### 通过nodePort的暴露服务

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


### 通过ingress暴露服务


上面用的是ndePort的方式，不方便管理，而且要在每台机器上都开一个port，也不好。通过ingress的方式可以解决这个问题。

jobmanager-ingress.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-flink
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: app.host.local
    http:
      paths:
      - backend:
          serviceName: flink-jobmanager
          servicePort: 8081
        path: /app(/|$)(.*)

```
就可以通过app.host.local/app/来访问了。一定要带app/的后斜杠访问。


### ingress方式下提交作业

#### 通过FQDN的方式

那么提交作业呢，就没办法通过域名的restApi访问了，因为加了url进行了重定向，原生的api只有域名和端口。可以通过如下方式：


1. 获取coreDns的ip

```
kubectl get pods -n kube-system | grep dns
coredns-7f9c544f75-p4rvz               1/1     Running   1          6d5h
coredns-7f9c544f75-psbnx               1/1     Running   1          6d5h
```

```
kubectl describe pod coredns-7f9c544f75-p4rvz -n kube-system

...
Annotations:          <none>
Status:               Running
IP:                   10.32.0.2
...

```

修改本机的dns解析

```
cat /etc/resolv.conf 
nameserver 10.32.0.2
nameserver 119.29.29.29
nameserver 114.114.114.114
options randomize-case:0
options timeout:2

```


可以通过
<service-name>.<namespace-name>.svc.cluster.local:8081
的方式访问clusterIP形式的服务了。


例如

```
/usr/local/flink/flink-1.10.0/bin/flink run -p  30 -m flink-jobmanager-rest.flink-project.svc.cluster.local:8081 -d -c test.startClass  Porject.jar
```
提交jar包


#### 通过脚本获取clusterIP的方式与jobManager

##### 以参数的形式提交


```
sh /opt/flink-submit.sh -n flink-project -u /usr/local/flink/flink-1.10.0/ -c study.Starter -j Study.jar -p 200
```

flink-submit.sh

```
#!/bin/bash

function usage() {
      cat << EOF
Usage: $arg0 options

OPTIONS:
  -n namespace        Kubernetes namespace
  -h                  Print usage info
  -c class            main class of the jar
  -u flink path       the path of flink
  -p parallelism      parallelism  
  

Example: $arg0 -n flink-link-analyse-search
EOF
}

while getopts 'n:c:j:u:p:h' OPT; do
    case $OPT in
        n)
            NAMESPACE="${OPTARG#=}"
            ;;
        c) 
            CLASS="${OPTARG#=}"
            ;;
        j)
            JARPATH="${OPTARG#=}"
            ;;
        u)  
            FLINKPATH="${OPTARG#=}" 
            ;;
        p)
            PARALLELISM="${OPTARG#=}"
            ;;
        h)
            usage;;
        ?)
            usage;;
    esac
done

if [[ $OPTIND -eq "1" ]]; then
    >&2 echo "No options were passed"
    usage
    exit 1
fi

shift $(( OPTIND - 1))

# Check NAMESPACE
if [ -z "$NAMESPACE" ];then
    >&2 echo "namespace miss"
    usage
    exit 1
fi

# Check FLINKPATH
if [ -z "$FLINKPATH" ];then
    >&2 echo "flink path miss"
    usage
    exit 1
fi


# Check MAIN CLASS
if [ -z "$CLASS" ];then
    >&2 echo "main class miss"
    usage
    exit 1
fi


# Check jar path
if [ -z "$JARPATH" ];then
    >&2 echo "jar path miss"
    usage
    exit 1
fi

# Check PARALLELISM
if [ -z "$PARALLELISM" ];then
    >&2 echo "parallelism miss"
    usage
    exit 1
fi



CLUSTERIP=$(kubectl get svc -n "$NAMESPACE"  -o=jsonpath='{.items[0].spec.clusterIP}')
echo namespace is $NAMESPACE
echo flink path is $FLINKPATH
echo jar is $JARPATH
echo main class is $CLASS
echo parallelism is $PARALLELISM
$FLINKPATH"/bin/flink" run -p $PARALLELISM  -m $CLUSTERIP:8081 -d -c $CLASS  $JARPATH

```

#### 以配置文件的形式

通过
```
sh /opt/flink-submit-config.sh
```
提交作业

通过
```
sh /opt/flink-list.sh
```
来获取作业运行信息

```
sh /opt/flink-cancel.sh -j 03e80ff36f00b761eb81400b6ae8328d
```
取消作业



首先，当前目录下比较要有flink-config.yaml文件

```
namespace: flink-project
class: study.Starter
flink.path: /usr/local/flink/flink-1.10.0/ 
parallelism: 200
jar.path: study.Starter
```

上述的脚本都会调用 /opt/flink-config.sh获取配置信息，然后作为变量。


flink-config.sh

```
readFromConfig() {
    local key=$1
    local defaultValue=$2
    local configFile=$3

    # first extract the value with the given key (1st sed), then trim the result (2nd sed)
    # if a key exists multiple times, take the "last" one (tail)
    local value=`sed -n "s/^[ ]*${key}[ ]*: \([^#]*\).*$/\1/p" "${configFile}" | sed "s/^ *//;s/ *$//" | tail -n 1`

    [ -z "$value" ] && echo "$defaultValue" || echo "$value"
}

NAMESPACE=$(readFromConfig namespace "" flink-config.yaml)
FLINKPATH=$(readFromConfig flink.path "" flink-config.yaml)
PARALLELISM=$(readFromConfig parallelism 1 flink-config.yaml)
JARPATH=$(readFromConfig jar.path "" flink-config.yaml)
CLASS=$(readFromConfig class "" flink-config.yaml)


```


/opt/flink-submit-config.sh

```shell
#!/bin/bash

function usage() {
      cat << EOF
Usage: $arg0 options

OPTIONS:
  -n namespace        Kubernetes namespace
  -h                  Print usage info
  -c class            main class of the jar
  -u flink path       the path of flink
  -p parallelism      parallelism  
  

Example: $arg0 -n flink-link-analyse-search
EOF
}

. /opt/flink-config.sh


# Check NAMESPACE
if [ -z "$NAMESPACE" ];then
    >&2 echo "namespace miss"
    usage
    exit 1
fi

# Check FLINKPATH
if [ -z "$FLINKPATH" ];then
    >&2 echo "flink path miss"
    usage
    exit 1
fi


# Check MAIN CLASS
if [ -z "$CLASS" ];then
    >&2 echo "main class miss"
    usage
    exit 1
fi


# Check jar path
if [ -z "$JARPATH" ];then
    >&2 echo "jar path miss"
    usage
    exit 1
fi

# Check PARALLELISM
if [ -z "$PARALLELISM" ];then
    >&2 echo "parallelism miss"
    usage
    exit 1
fi



CLUSTERIP=$(kubectl get svc -n "$NAMESPACE"  -o=jsonpath='{.items[0].spec.clusterIP}')
echo namespace is $NAMESPACE
echo flink path is $FLINKPATH
echo jar is $JARPATH
echo main class is $CLASS
echo parallelism is $PARALLELISM
echo $FLINKPATH"/bin/flink" run -p $PARALLELISM  -m $CLUSTERIP:8081 -d -c $CLASS  $JARPATH
$FLINKPATH"/bin/flink" run -p $PARALLELISM  -m $CLUSTERIP:8081 -d -c $CLASS  $JARPATH

```

/opt/flink-list.sh 
```shell

. /opt/flink-config.sh

# Check NAMESPACE
if [ -z "$NAMESPACE" ];then
    >&2 echo "namespace miss"
    usage
    exit 1
fi

# Check FLINKPATH
if [ -z "$FLINKPATH" ];then
    >&2 echo "flink path miss"
    usage
    exit 1
fi


CLUSTERIP=$(kubectl get svc -n "$NAMESPACE"  -o=jsonpath='{.items[0].spec.clusterIP}')
echo namespace is $NAMESPACE
echo flink path is $FLINKPATH
echo $FLINKPATH"/bin/flink" list   -m $CLUSTERIP:8081
$FLINKPATH"/bin/flink" list   -m $CLUSTERIP:8081 
```

/opt/flink-cancel.sh 

```shell
. /opt/flink-config.sh

function usage() {
      cat << EOF
Usage: $arg0 options

OPTIONS:
  -j job id       
  -h                  Print usage info
  

EOF
}

while getopts 'j:h' OPT; do
    case $OPT in
        j)
            JOBID="${OPTARG#=}"
            ;;
        h)
            usage;;
        ?)
            usage;;
    esac
done

# Check NAMESPACE
if [ -z "$NAMESPACE" ];then
    >&2 echo "namespace miss"
    usage
    exit 1
fi

# Check FLINKPATH
if [ -z "$JOBID" ];then
    >&2 echo "job id miss"
    usage
    exit 1
fi


CLUSTERIP=$(kubectl get svc -n "$NAMESPACE"  -o=jsonpath='{.items[0].spec.clusterIP}')
echo namespace is $NAMESPACE
echo jobid is $JOBID
echo $FLINKPATH"/bin/flink" cancel $JOBID   -m $CLUSTERIP:8081
$FLINKPATH"/bin/flink" cancel $JOBID   -m $CLUSTERIP:8081 
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

### config比较完整的配置
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: flink-config
  labels:
    app: flink
data:
  flink-conf.yaml: |+
    jobmanager.rpc.address: flink-jobmanager
    taskmanager.numberOfTaskSlots: 3
    blob.server.port: 6124
    jobmanager.rpc.port: 6123
    taskmanager.rpc.port: 6122
    taskmanager.memory.process.size: 12000m
    jobmanager.heap.size: 1024m
    jobmanager.execution.failover-strategy: full
    jobstore.expiration-time: 172800
    taskmanager.memory.managed.fraction: 0.2
    taskmanager.network.memory.min: 2gb
    taskmanager.network.memory.max: 3gb
    taskmanager.memory.task.off-heap.size: 1024m
    taskmanager.network.memory.floating-buffers-per-gate: 16
    taskmanager.network.memory.buffers-per-channel: 4
    taskmanager.memory.jvm-metaspace.size: 256m
    akka.ask.timeout: 30s
    akka.framesize: 104857600b
    restart-strategy: failure-rate
    restart-strategy.failure-rate.max-failures-per-interval: 300
    restart-strategy.failure-rate.failure-rate-interval: 5 min
    restart-strategy.failure-rate.delay: 10 s
    state.checkpoints.dir: hdfs://hdp.host.local:9000/flink/flink-checkpoints
    state.savepoints.dir: hdfs://hdp.host.local:9000/flink/flink-savepoints
    state.backend: rocksdb
    state.backend.incremental: true
    high-availability: zookeeper
    high-availability.storageDir: hdfs://hdp.host.local:9000/flink/recovery
    high-availability.zookeeper.quorum: zk.host.local:7072
    high-availability.zookeeper.path.root: /flink
    high-availability.jobmanager.port: 6123
    high-availability.cluster-id: my-project
    metrics.reporter.promgateway.class: org.apache.flink.metrics.prometheus.PrometheusPushGatewayReporter
    metrics.reporter.promgateway.host: pushgateway.host.local
    metrics.reporter.promgateway.port: 30002
    metrics.reporter.promgateway.jobName: my-project
    metrics.reporter.promgateway.randomJobNameSuffix: true
    metrics.reporter.promgateway.deleteOnShutdown: true
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

### 资源限制


一般而言，我们会在资源上做限制，避免flink跑高互相影响，但是如果出现超过资源上限，是会被kill的，问题也不好查。

```
    resources: 
         requests:
          cpu: "5000m"
          memory: "42720Mi"
         limits: 
          cpu: "5000m"
          memory: "42720Mi"

```


### rockersdb写磁盘

由于状态后台很吃磁盘性能，直接用系统盘会有性能问题。通过hostPath将磁盘挂载到指定目录



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
        image: k8stest.net:8876/study/flink:1.10.0
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
        - name: state-volume
          mountPath : /opt/state
      volumes:
      - name: flink-config-volume
        configMap:
          name: flink-config
          items:
          - key: flink-conf.yaml
            path: flink-conf.yaml
          - key: log4j.properties
            path: log4j.properties
      - name: state-volume
        hostPath: 
          path: /cache1/flink/state
          type: DirectoryOrCreate


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
        image: k8stest.net:8876/study/flink:1.10.0
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
        - name: state-volume
          mountPath : /opt/state
      volumes:
      - name: flink-config-volume
        configMap:
          name: flink-config
          items:
          - key: flink-conf.yaml
            path: flink-conf.yaml
          - key: log4j.properties
            path: log4j.properties
      - name: state-volume
        hostPath: 
          path: /cache1/flink/state
          type: DirectoryOrCreate
```


在配置文件中配置

```
state.backend.rocksdb.localdir: /opt/state
```


### Reference 



https://ci.apache.org/projects/flink/flink-docs-release-1.9/ops/deployment/kubernetes.html


HA:
http://shzhangji.com/cnblogs/2019/08/25/deploy-flink-job-cluster-on-kubernetes/
