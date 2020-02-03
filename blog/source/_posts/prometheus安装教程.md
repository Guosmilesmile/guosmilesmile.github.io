---
title: prometheus安装教程
date: 2020-01-22 12:37:20
tags:
categories:
	- Kubernetes
---
### 配置文件

为了能够方便的管理配置文件，我们这里将 prometheus.yml 文件用 ConfigMap 的形式进行管理：（prometheus-cm.yaml）

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: kube-ops
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 15s
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
```
我们这里暂时只配置了对 prometheus 的监控，然后创建该资源对象：
```
$ kubectl create -f prometheus-cm.yaml
configmap "prometheus-config" created
```
配置文件创建完成了，以后如果我们有新的资源需要被监控，我们只需要将上面的 ConfigMap 对象更新即可。现在我们来创建 prometheus 的 Pod 资源：(prometheus-deploy.yaml)

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: prometheus
  namespace: kube-ops
  labels:
    app: prometheus
spec:
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - image: prom/prometheus:v2.4.3
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention=24h"
        - "--web.enable-admin-api"  # 控制对admin HTTP API的访问，其中包括删除时间序列等功能
        - "--web.enable-lifecycle"  # 支持热更新，直接执行localhost:9090/-/reload立即生效
        ports:
        - containerPort: 9090
          protocol: TCP
          name: http
        volumeMounts:
        - mountPath: "/prometheus"
          subPath: prometheus
          name: data
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
          limits:
            cpu: 100m
            memory: 512Mi
      securityContext:
        runAsUser: 0
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: prometheus
      - configMap:
          name: prometheus-config
        name: config-volume
```

我们在启动程序的时候，除了指定了 prometheus.yml 文件之外，还通过参数storage.tsdb.path指定了 TSDB 数据的存储路径、通过storage.tsdb.retention设置了保留多长时间的数据，还有下面的web.enable-admin-api参数可以用来开启对 admin api 的访问权限，参数web.enable-lifecycle非常重要，用来开启支持热更新的，有了这个参数之后，prometheus.yml 配置文件只要更新了，通过执行localhost:9090/-/reload就会立即生效，所以一定要加上这个参数。

我们这里将 prometheus.yml 文件对应的 ConfigMap 对象通过 volume 的形式挂载进了 Pod，这样 ConfigMap 更新后，对应的 Pod 里面的文件也会热更新的，然后我们再执行上面的 reload 请求，Prometheus 配置就生效了，除此之外，为了将时间序列数据进行持久化，我们将数据目录和一个 pvc 对象进行了绑定，所以我们需要提前创建好这个 pvc 对象：(prometheus-volume.yaml)


```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.151.30.57
    path: /data/k8s

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus
  namespace: kube-ops
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

如果不适用nfs，可以直接指定一台宿主机，然后设置为hostPath


```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  hostPath:
    path: /cache1/grafana/data/

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus
  namespace: kube-ops
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

```

```
$ kubectl create -f prometheus-volume.yaml
```

除了上面的注意事项外，我们这里还需要配置 rbac 认证，因为我们需要在 prometheus 中去访问 Kubernetes 的相关信息，所以我们这里管理了一个名为 prometheus 的 serviceAccount 对象：(prometheus-rbac.yaml)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: kube-ops
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: kube-ops
```

由于我们要获取的资源信息，在每一个 namespace 下面都有可能存在，所以我们这里使用的是 ClusterRole 的资源对象，值得一提的是我们这里的权限规则声明中有一个nonResourceURLs的属性，是用来对非资源型 metrics 进行操作的权限声明，这个在以前我们很少遇到过，然后直接创建上面的资源对象即可：

```
$ kubectl create -f prometheus-rbac.yaml
serviceaccount "prometheus" created
clusterrole.rbac.authorization.k8s.io "prometheus" created
clusterrolebinding.rbac.authorization.k8s.io "prometheus" created
```

还有一个要注意的地方是我们这里必须要添加一个securityContext的属性，将其中的runAsUser设置为0，这是因为现在的 prometheus 运行过程中使用的用户是 nobody，否则会出现下面的permission denied之类的权限错误：

```
level=error ts=2018-10-22T14:34:58.632016274Z caller=main.go:617 err="opening storage failed: lock DB directory: open /data/lock: permission denied"
```

现在我们就可以添加 promethues 的资源对象了：

```
$ kubectl create -f prometheus-deploy.yaml
deployment.extensions "prometheus" created
$ kubectl get pods -n kube-ops
NAME                          READY     STATUS    RESTARTS   AGE
prometheus-6dd775cbff-zb69l   1/1       Running   0          20m
$ kubectl logs -f prometheus-6dd775cbff-zb69l -n kube-ops
......
level=info ts=2018-10-22T14:44:40.535385503Z caller=main.go:523 msg="Server is ready to receive web requests."
```
Pod 创建成功后，为了能够在外部访问到 prometheus 的 webui 服务，我们还需要创建一个 Service 对象：(prometheus-svc.yaml)

```
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: kube-ops
  labels:
    app: prometheus
spec:
  selector:
    app: prometheus
  type: NodePort
  ports:
    - name: web
      port: 9090
      targetPort: http
      nodePode: 30000
```
然后我们就可以通过http://任意节点IP:30000访问 prometheus 的 webui 服务了。





### Reference 

https://www.qikqiak.com/k8s-book/docs/52.Prometheus%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8.html