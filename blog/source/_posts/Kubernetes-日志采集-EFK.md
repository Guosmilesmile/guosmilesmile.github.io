---
title: Kubernetes 日志采集 EFK
date: 2020-05-09 20:44:14
tags:
categories:
	- Kubernetes
---



## 日志采集的必要性

容器中的日志，在容器销毁后会跟着容器一起消失，很多时候，程序报错导致的容器重启会带走之前的报错日志，只有通过ELK的形式将日志采集带外部，才可以进行追诉。


本次选用EFK来进行日志的采集，通过fluentd将日志直接发送到es中，然后对接到kibana展示。


## 日志采集过程

采集过程简单说就是利用 Fluentd 采集 Kubernetes 节点服务器的 “/var/log” 和 “/var/lib/docker/container” 两个目录下的日志信息，然后汇总到 ElasticSearch 集群中，再经过 Kibana 展示的一个过程。


具体日志收集过程如下所述：

1. 创建 Fluentd 并且将 Kubernetes 节点服务器 log 目录挂载进容器;
2. Fluentd 启动采集 log 目录下的 containers 里面的日志文件;
3. Fluentd 将收集的日志转换成 JSON 格式;
4. Fluentd 利用 Exception Plugin 检测日志是否为容器抛出的异常日志，是就将异常栈的多行日志合并;
5. Fluentd 将换行多行日志 JSON 合并;
6. Fluentd 使用 Kubernetes Metadata Plugin 检测出 Kubernetes 的 Metadata 数据进行过滤，如 Namespace、Pod Name等；
7. Fluentd 使用 ElasticSearch 插件将整理完的 JSON 日志输出到 ElasticSearch 中;
8. ElasticSearch 建立对应索引，持久化日志信息。
9. Kibana 检索 ElasticSearch 中 Kubernetes 日志相关信息进行展示。


## 简单日志收集过程图

![image](https://note.youdao.com/yws/api/personal/file/DBE21B61EC76418CAEC04A3FEAFF6F82?method=download&shareKey=4d3f41d90c03829b9460a21bf1a84451)

## 安装过程

### 准备 Fluentd 配置文件

详情请访问 Kubernetes Fluentd

Github地址：https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch


## 配置文件分析


```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: fluentd-es-config-v0.2.0
  namespace: kube-system
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
  ###### 系统配置，默认即可 #######
  system.conf: |-
    <system>
      root_dir /tmp/fluentd-buffers/
    </system>
    
  ###### 容器日志—收集配置 #######    
  containers.input.conf: |-
    # ------采集 Kubernetes 容器日志-------
    <source>                                  
      @id fluentd-containers.log
      @type tail                              #---Fluentd 内置的输入方式，其原理是不停地从源文件中获取新的日志。
      path /var/log/containers/*.log          #---挂载的服务器Docker容器日志地址
      pos_file /var/log/es-containers.log.pos
      tag raw.kubernetes.*                    #---设置日志标签
      read_from_head true
      <parse>                                 #---多行格式化成JSON
        @type multi_format                    #---使用multi-format-parser解析器插件
        <pattern>
          format json                         #---JSON解析器
          time_key time                       #---指定事件时间的时间字段
          time_format %Y-%m-%dT%H:%M:%S.%NZ   #---时间格式
        </pattern>
        <pattern>
          format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          time_format %Y-%m-%dT%H:%M:%S.%N%:z
        </pattern>
      </parse>
    </source>
    
    # -----检测Exception异常日志连接到一条日志中------
    # 关于插件请查看地址：https://github.com/GoogleCloudPlatform/fluent-plugin-detect-exceptions
    <match raw.kubernetes.**>     #---匹配tag为raw.kubernetes.**日志信息
      @id raw.kubernetes
      @type detect_exceptions     #---使用detect-exceptions插件处理异常栈信息，放置异常只要一行而不完整
      remove_tag_prefix raw       #---移出raw前缀
      message log                 #---JSON记录中包含应扫描异常的单行日志消息的字段的名称。
                                  #   如果将其设置为''，则插件将按此顺序尝试'message'和'log'。
                                  #   此参数仅适用于结构化（JSON）日志流。默认值：''。
      stream stream               #---JSON记录中包含“真实”日志流中逻辑日志流名称的字段的名称。
                                  #   针对每个逻辑日志流单独处理异常检测，即，即使逻辑日志流 的
                                  #   消息在“真实”日志流中交织，也将检测到异常。因此，仅组合相
                                  #   同逻辑流中的记录。如果设置为''，则忽略此参数。此参数仅适用于
                                  #   结构化（JSON）日志流。默认值：''。
      multiline_flush_interval 5  #---以秒为单位的间隔，在此之后将转发（可能尚未完成）缓冲的异常堆栈。
                                  #   如果未设置，则不刷新不完整的异常堆栈。
      max_bytes 500000
      max_lines 1000
    </match>
  
    # -------日志拼接-------
    <filter **>
      @id filter_concat
      @type concat                #---Fluentd Filter插件，用于连接多个事件中分隔的多行日志。
      key message
      multiline_end_regexp /\n$/  #---以换行符“\n”拼接
      separator ""
    </filter> 
  
    # ------过滤Kubernetes metadata数据使用pod和namespace metadata丰富容器日志记录-------
    # 关于插件请查看地址：https://github.com/fabric8io/fluent-plugin-kubernetes_metadata_filter
    <filter kubernetes.**>
      @id filter_kubernetes_metadata
      @type kubernetes_metadata
    </filter>
    
    # ------修复ElasticSearch中的JSON字段------
    # 关于插件请查看地址：https://github.com/repeatedly/fluent-plugin-multi-format-parser
    <filter kubernetes.**>
      @id filter_parser
      @type parser                #---multi-format-parser多格式解析器插件
      key_name log                #---在要解析的记录中指定字段名称。
      reserve_data true           #---在解析结果中保留原始键值对。
      remove_key_name_field true  #---key_name解析成功后删除字段。
      <parse>
        @type multi_format
        <pattern>
          format json
        </pattern>
        <pattern>
          format none
        </pattern>
      </parse>
    </filter>
    
  ###### Kuberntes集群节点机器上的日志收集 ######    
  system.input.conf: |-
    # ------Kubernetes minion节点日志信息，可以去掉------
    #<source>
    #  @id minion
    #  @type tail
    #  format /^(?<time>[^ ]* [^ ,]*)[^\[]*\[[^\]]*\]\[(?<severity>[^ \]]*) *\] (?<message>.*)$/
    #  time_format %Y-%m-%d %H:%M:%S
    #  path /var/log/salt/minion
    #  pos_file /var/log/salt.pos
    #  tag salt
    #</source>
    
    # ------启动脚本日志，可以去掉------
    # <source>
    #   @id startupscript.log
    #   @type tail
    #   format syslog
    #   path /var/log/startupscript.log
    #   pos_file /var/log/es-startupscript.log.pos
    #   tag startupscript
    # </source>
    
    # ------Docker 程序日志，可以去掉------
    # <source>
    #   @id docker.log
    #   @type tail
    #   format /^time="(?<time>[^)]*)" level=(?<severity>[^ ]*) msg="(?<message>[^"]*)"( err="(?<error>[^"]*)")?( statusCode=($<status_code>\d+))?/
    #   path /var/log/docker.log
    #   pos_file /var/log/es-docker.log.pos
    #   tag docker
    # </source>
    
    #------ETCD 日志，因为ETCD现在默认启动到容器中，采集容器日志顺便就采集了，可以去掉------
    # <source>
    #   @id etcd.log
    #   @type tail
    #   # Not parsing this, because it doesn't have anything particularly useful to
    #   # parse out of it (like severities).
    #   format none
    #   path /var/log/etcd.log
    #   pos_file /var/log/es-etcd.log.pos
    #   tag etcd
    # </source>
    
    #------Kubelet 日志------
    # <source>
    #   @id kubelet.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/kubelet.log
    #   pos_file /var/log/es-kubelet.log.pos
    #   tag kubelet
    # </source>
    
    #------Kube-proxy 日志------
    # <source>
    #   @id kube-proxy.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/kube-proxy.log
    #   pos_file /var/log/es-kube-proxy.log.pos
    #   tag kube-proxy
    # </source>
    
    #------kube-apiserver日志------
    # <source>
    #   @id kube-apiserver.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/kube-apiserver.log
    #   pos_file /var/log/es-kube-apiserver.log.pos
    #   tag kube-apiserver
    # </source>

    #------Kube-controller日志------
    # <source>
    #   @id kube-controller-manager.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/kube-controller-manager.log
    #   pos_file /var/log/es-kube-controller-manager.log.pos
    #   tag kube-controller-manager
    # </source>
    
    #------Kube-scheduler日志------
    # <source>
    #   @id kube-scheduler.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/kube-scheduler.log
    #   pos_file /var/log/es-kube-scheduler.log.pos
    #   tag kube-scheduler
    # </source>
    
    #------glbc日志------
    # <source>
    #   @id glbc.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/glbc.log
    #   pos_file /var/log/es-glbc.log.pos
    #   tag glbc
    # </source>
    
    #------Kubernetes 伸缩日志------
    # <source>
    #   @id cluster-autoscaler.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/cluster-autoscaler.log
    #   pos_file /var/log/es-cluster-autoscaler.log.pos
    #   tag cluster-autoscaler
    # </source>

    # -------来自system-journal的日志------
    <source>
      @id journald-docker
      @type systemd
      matches [{ "_SYSTEMD_UNIT": "docker.service" }]
      <storage>
        @type local
        persistent true
        path /var/log/journald-docker.pos
      </storage>
      read_from_head true
      tag docker
    </source>

    # -------Journald-container-runtime日志信息------
    <source>
      @id journald-container-runtime
      @type systemd
      matches [{ "_SYSTEMD_UNIT": "{{ fluentd_container_runtime_service }}.service" }]
      <storage>
        @type local
        persistent true
        path /var/log/journald-container-runtime.pos
      </storage>
      read_from_head true
      tag container-runtime
    </source>
    
    # -------Journald-kubelet日志信息------
    <source>
      @id journald-kubelet
      @type systemd
      matches [{ "_SYSTEMD_UNIT": "kubelet.service" }]
      <storage>
        @type local
        persistent true
        path /var/log/journald-kubelet.pos
      </storage>
      read_from_head true
      tag kubelet
    </source>
    
    # -------journald节点问题检测器------
    #关于插件请查看地址：https://github.com/reevoo/fluent-plugin-systemd
    #systemd输入插件，用于从systemd日志中读取日志
    <source>
      @id journald-node-problem-detector
      @type systemd
      matches [{ "_SYSTEMD_UNIT": "node-problem-detector.service" }]
      <storage>
        @type local
        persistent true
        path /var/log/journald-node-problem-detector.pos
      </storage>
      read_from_head true
      tag node-problem-detector
    </source>
    
    # -------kernel日志------ 
    <source>
      @id kernel
      @type systemd
      matches [{ "_TRANSPORT": "kernel" }]
      <storage>
        @type local
        persistent true
        path /var/log/kernel.pos
      </storage>
      <entry>
        fields_strip_underscores true
        fields_lowercase true
      </entry>
      read_from_head true
      tag kernel
    </source>
    
  ###### 监听配置，一般用于日志聚合用 ######
  forward.input.conf: |-
    #监听通过TCP发送的消息
    <source>
      @id forward
      @type forward
    </source>

  ###### Prometheus metrics 数据收集 ######
  monitoring.conf: |-
    # input plugin that exports metrics
    # 输出 metrics 数据的 input 插件
    <source>
      @id prometheus
      @type prometheus
    </source>
    <source>
      @id monitor_agent
      @type monitor_agent
    </source>
    # 从 MonitorAgent 收集 metrics 数据的 input 插件 
    <source>
      @id prometheus_monitor
      @type prometheus_monitor
      <labels>
        host ${hostname}
      </labels>
    </source>
    # ------为 output 插件收集指标的 input 插件------
    <source>
      @id prometheus_output_monitor
      @type prometheus_output_monitor
      <labels>
        host ${hostname}
      </labels>
    </source>
    # ------为in_tail 插件收集指标的input 插件------
    <source>
      @id prometheus_tail_monitor
      @type prometheus_tail_monitor
      <labels>
        host ${hostname}
      </labels>
    </source>

  ###### 输出配置，在此配置输出到ES的配置信息 ######
  # ElasticSearch fluentd插件地址：https://docs.fluentd.org/v1.0/articles/out_elasticsearch
  output.conf: |-
    <match **>
      @id elasticsearch
      @type elasticsearch
      @log_level info     #---指定日志记录级别。可设置为fatal，error，warn，info，debug，和trace，默认日志级别为info。
      type_name _doc
      include_tag_key true              #---将 tag 标签的 key 到日志中。
      host elasticsearch-logging        #---指定 ElasticSearch 服务器地址。
      port 9200                         #---指定 ElasticSearch 端口号。
      #index_name fluentd.${tag}.%Y%m%d #---要将事件写入的索引名称（默认值:) fluentd。
      logstash_format true              #---使用传统的索引名称格式logstash-%Y.%m.%d，此选项取代该index_name选项。
      #logstash_prefix logstash         #---用于logstash_format指定为true时写入logstash前缀索引名称，默认值:logstash。
      <buffer>
        @type file                      #---Buffer 插件类型，可选file、memory
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff  #---重试模式，可选为exponential_backoff、periodic。
                                        #   exponential_backoff 模式为等待秒数，将在每次失败时成倍增长
        flush_thread_count 2
        flush_interval 10s
        retry_forever
        retry_max_interval 30           #---丢弃缓冲数据之前的尝试的最大间隔。
        chunk_limit_size 5M             #---每个块的最大大小：事件将被写入块，直到块的大小变为此大小。
        queue_limit_length 8            #---块队列的长度。
        overflow_action block           #---输出插件在缓冲区队列已满时的行为方式，有throw_exception、block、
                                        #   drop_oldest_chunk，block方式为阻止输入事件发送到缓冲区。
      </buffer>
    </match>
```

### 定制配置并调整参数

暂时不需要采集系统日志，把系统日志部分去掉

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: fluentd-es-config
  namespace: logging
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
  #------系统配置参数-----
  system.conf: |-
    <system>
      root_dir /tmp/fluentd-buffers/
    </system>
  #------Kubernetes 容器日志收集配置------
  containers.input.conf: |-
    <source>
      @id fluentd-containers.log
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/es-containers.log.pos
      tag raw.kubernetes.*
      read_from_head true
      <parse>
        @type multi_format
        <pattern>
          format json
          time_key time
          time_format %Y-%m-%dT%H:%M:%S.%NZ
        </pattern>
        <pattern>
          format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          time_format %Y-%m-%dT%H:%M:%S.%N%:z
        </pattern>
      </parse>
    </source>
    <match raw.kubernetes.**>
      @id raw.kubernetes
      @type detect_exceptions
      remove_tag_prefix raw
      message log
      stream stream
      multiline_flush_interval 5
      max_bytes 500000
      max_lines 1000
    </match>
    <filter **>
      @id filter_concat
      @type concat
      key message
      multiline_end_regexp /\n$/
      separator ""
    </filter>
    <filter kubernetes.**>
      @id filter_kubernetes_metadata
      @type kubernetes_metadata
    </filter>
    <filter kubernetes.**>
      @id filter_parser
      @type parser
      key_name log
      reserve_data true
      remove_key_name_field true
      <parse>
        @type multi_format
        <pattern>
          format json
        </pattern>
        <pattern>
          format none
        </pattern>
      </parse>
    </filter>
  #------系统日志收集-------
  system.input.conf: |-  
    <source>
      @id journald-container-runtime
      @type systemd
      matches [{ "_SYSTEMD_UNIT": "{{ fluentd_container_runtime_service }}.service" }]
      <storage>
        @type local
        persistent true
        path /var/log/journald-container-runtime.pos
      </storage>
      read_from_head true
      tag container-runtime
    </source>
    <source>
      @id journald-node-problem-detector
      @type systemd
      matches [{ "_SYSTEMD_UNIT": "node-problem-detector.service" }]
      <storage>
        @type local
        persistent true
        path /var/log/journald-node-problem-detector.pos
      </storage>
      read_from_head true
      tag node-problem-detector
    </source>
  #------输出到 ElasticSearch 配置------
  output.conf: |-
    <match **>
      @id elasticsearch
      @type elasticsearch
      @log_level info
      type_name _doc
      include_tag_key true
      host elasticsearch-client     #改成自己的 ElasticSearch 地址
      port 9200
      logstash_format true
      logstash_prefix kubernetes   # index 前缀
      logst
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_thread_count 5
        flush_interval 8s
        retry_forever
        retry_max_interval 30
        chunk_limit_size 5M
        queue_limit_length 10
        overflow_action block
        compress gzip               #开启gzip提高日志采集性能
      </buffer>
    </match>
```

### DaemonSet文件

https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/fluentd-elasticsearch/fluentd-es-ds.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd-es
  namespace: kube-system
  labels:
    k8s-app: fluentd-es
    addonmanager.kubernetes.io/mode: Reconcile
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    addonmanager.kubernetes.io/mode: Reconcile
rules:
- apiGroups:
  - ""
  resources:
  - "namespaces"
  - "pods"
  verbs:
  - "get"
  - "watch"
  - "list"
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    addonmanager.kubernetes.io/mode: Reconcile
subjects:
- kind: ServiceAccount
  name: fluentd-es
  namespace: kube-system
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: fluentd-es
  apiGroup: ""
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-es-v3.0.1
  namespace: kube-system
  labels:
    k8s-app: fluentd-es
    version: v3.0.1
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-es
      version: v3.0.1
  template:
    metadata:
      labels:
        k8s-app: fluentd-es
        version: v3.0.1
      # This annotation ensures that fluentd does not get evicted if the node
      # supports critical pod annotation based priority scheme.
      # Note that this does not guarantee admission on the nodes (#40573).
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
    spec:
      priorityClassName: system-node-critical
      serviceAccountName: fluentd-es
      containers:
      - name: fluentd-es
        image: quay.io/fluentd_elasticsearch/fluentd:v3.0.1
        env:
        - name: FLUENTD_ARGS
          value: --no-supervisor -q
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config-volume
          mountPath: /etc/fluent/config.d
        ports:
        - containerPort: 24231
          name: prometheus
          protocol: TCP
        livenessProbe:
          tcpSocket:
            port: prometheus
          initialDelaySeconds: 5
          timeoutSeconds: 10
        readinessProbe:
          tcpSocket:
            port: prometheus
          initialDelaySeconds: 5
          timeoutSeconds: 10
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config-volume
        configMap:
          name: fluentd-es-config-v0.2.0

```



将上面的两个yaml文件部署在kubernetes集群。

```
kubectl apply -f fluentd.yaml
kubectl apply -f fluentd-es-config.yaml
```

## Kibana 查看采集的日志信息


### Kibana 设置索引

#### Kibana 中设置检索的索引


![image](https://note.youdao.com/yws/api/personal/file/69479F08485D4409977C3B107B063843?method=download&shareKey=97b1334cd5377365957033889e4557ff)

![image](https://note.youdao.com/yws/api/personal/file/C58B010BFC42434E9857EA8CC1DE2834?method=download&shareKey=fcf23a0b2420a166c00b992143d21481)


由于在 Fluentd 输出配置中配置了 “logstash_prefix kubernetes” 这个参数，所以索引是以 kubernetes 为前缀显示，如果未设置则默认为 “logstash” 为前缀。

![image](https://note.youdao.com/yws/api/personal/file/21AC1B2FFECD4715BFA2571C3FF17883?method=download&shareKey=328b93e15d5deaa92d300b074457df10)

![image](https://note.youdao.com/yws/api/personal/file/CBCC60EB825B4FE49C6FBFB1C61ABCE7?method=download&shareKey=c0b0adf647d7a6adf2b835c803f034c3)

![image](https://note.youdao.com/yws/api/personal/file/1F5E192DB36F4E0FBE7DF56E84234E16?method=download&shareKey=24745e866292170dc04b6499a1d4a347)

## Reference

[Kubernetes 日志采集 EFK](http://www.mydlq.club/article/14/)
[Fluentd官方文档](https://docs.fluentd.org/v/0.12/)
[kubernetes github安装文档](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch)
