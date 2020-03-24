---
title: kubernetes ingress
date: 2020-03-20 11:06:20
tags:
categories:
	- Kubernetes
---



### kubernetes 对外暴露服务的方法

向 kubernetes 集群外部暴露服务的方式有三种： nodePort，LoadBalancer 和本文要介绍的 Ingress。每种方式都有各自的优缺点，nodePort 方式在服务变多的情况下会导致节点要开的端口越来越多，不好管理。而 LoadBalancer 更适合结合云提供商的 LB 来使用，但是在 LB 越来越多的情况下对成本的花费也是不可小觑。Ingress 是 kubernetes 官方提供的用于对外暴露服务的方式，也是在生产环境用的比较多的方式，一般在云环境下是 LB + Ingress Ctroller 方式对外提供服务，这样就可以在一个 LB 的情况下根据域名路由到对应后端的 Service，有点类似于 Nginx 反向代理，只不过在 kubernetes 集群中，这个反向代理是集群外部流量的统一入口。



### Ingress 及 Ingress Controller 简介



Ingress 是 k8s 资源对象，用于对外暴露服务，该资源对象定义了不同主机名（域名）及 URL 和对应后端 Service（k8s Service）的绑定，根据不同的路径路由 http 和 https 流量。而 Ingress Contoller 是一个 pod 服务，封装了一个 web 前端负载均衡器，同时在其基础上实现了动态感知 Ingress 并根据 Ingress 的定义动态生成 前端 web 负载均衡器的配置文件，比如 Nginx Ingress Controller 本质上就是一个 Nginx，只不过它能根据 Ingress 资源的定义动态生成 Nginx 的配置文件，然后动态 Reload。个人觉得 Ingress Controller 的重大作用是将前端负载均衡器和 Kubernetes 完美地结合了起来，一方面在云、容器平台下方便配置的管理，另一方面实现了集群统一的流量入口，而不是像 nodePort 那样给集群打多个孔。



![在这里插入图片描述](https://img-blog.csdnimg.cn/20190812232401601.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpYW5naGFvaGFv,size_16,color_FFFFFF,t_70)


![img](https://img2018.cnblogs.com/blog/1216496/201901/1216496-20190129105538443-1113599925.png)



所以，总的来说要使用 Ingress，得先部署 Ingress Controller 实体（相当于前端 Nginx），然后再创建 Ingress （相当于 Nginx 配置的 k8s 资源体现），Ingress Controller 部署好后会动态检测 Ingress 的创建情况生成相应配置。Ingress Controller 的实现有很多种：有基于 Nginx 的，也有基于 HAProxy的，还有基于 OpenResty 的 Kong Ingress Controller 等，更多 Controller 见：https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/，本文使用基于 Nginx 的 Ingress Controller：ingress-nginx。





### Nginx Ingress Controller



官方文档参考：https://kubernetes.github.io/ingress-nginx/deploy/



首先部署命名空间，默认后端（处理 404 等），配置等，与官方一致。

https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml



官方的配置文件使用 Deployment 的方式部署单副本到任意一台 worker 机器，但我修改了一些配置，改变了以下行为：

1. Nginx 部署在 master 机器上，使用 master 的入口 ip 提供服务

2. 官方文档部署完后仍然需要使用 Service 做转发，在没有 ELB 的情况下仍需使用 NodePort 方式暴露在高端口上

   

```

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      # wait up to five minutes for the drain of connections
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      hostNetwork: true
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/master
                operator: Exists
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 101
            runAsUser: 101
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown

```



对官方配置的修改主要有以下几处：

1. 使用 `hostNetwork` 配置将服务暴露在外网接口
2. 使用[亲和性配置](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)限制服务只能部署在 master 上
3. 使用 `tolerations` 配置允许在 master 上部署此服务



因为使用了hostNetwork，直接绑定了宿主机的80和443端口了，不需要再使用NodePort的方式创建一个service。如果没有配置hstoNetwor，那么需要另外配置一个service。



全部配置见下方的配置清单



### 配置 Ingress



这个困扰我比较久，如果只是简单的80端口转发就搞定了，先上简单版本的吧。

```
kubectl get svc -n flink
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
flink-jobmanager        ClusterIP   10.104.134.45   <none>        6123/TCP,6124/TCP,8081/TCP   18h
flink-jobmanager-rest   ClusterIP   10.98.55.178    <none>        8081/TCP                     18h
```



先查看需要关联的服务，对应的名称是flink-jobmanager-rest，对应的端口是8081。



所以对应的ingress的资源文件应该如下配置



```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: submodule-checker-ingress-flink
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: flink.test.com
    http:
      paths:
      - backend:
          serviceName: flink-jobmanager-rest
          servicePort: 8081
        path: /
```



**有一点需要注意的是，ingress的资源文件需要跟service同一个空间，内部的匹配规则是 nameSpace.serviceName去匹配的。**



部署好以后，如果有公网域名最好了，如果没有就做一条本地host来模拟解析flink.test.com到node的ip地址。测试访问  http://flink.test.com/



但是这样80端口就会占用了，如果你有多个flink作业，想用区分开，如果想要通过后缀区分，例如

http://flink.test.com/app  , http://flink.test.com/app1 区分不同的作业，那么就需要进行重定向。



根据官网的重定向配置，配置文件应该修改为



```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: submodule-checker-ingress-flink
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /$2    
spec:
  rules:
  - host: flink.test.com
    http:
      paths:
      - backend:
          serviceName: flink-jobmanager-rest
          servicePort: 8081
        path: /app(/|$)(.*)
      

```



**注意了** 必须将地址输入全， http://flink.test.com/app 这样是访问不到的，必须是 http://flink.test.com/app/ 才可以正确访问到。



至此，可以通过ingress还有url来区分不同的资源了。



### 配置



ingress.yaml

```
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---


apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      # wait up to five minutes for the drain of connections
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      hostNetwork: true
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/master
                operator: Exists
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 101
            runAsUser: 101
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown


---

apiVersion: v1
kind: LimitRange
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  limits:
  - min:
      memory: 90Mi
      cpu: 100m
    type: Container

```



jobmanager-ingress.yaml

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: submodule-checker-ingress-flink
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /$2    
spec:
  rules:
  - host: flink.test.com
    http:
      paths:
      - backend:
          serviceName: flink-jobmanager-rest
          servicePort: 8081
        path: /app(/|$)(.*)
```



### Reference 

[官方安装教程](https://kubernetes.github.io/ingress-nginx/deploy/)

[官网配置--rewrite ](https://kubernetes.github.io/ingress-nginx/examples/rewrite/)

[在 Kubernetes 上部署 Nginx Ingress Cntroller]( https://blog.hlyue.com/2018/05/12/deploy-nginx-ingress-controller-on-kubernetes/)

[使用 Kubernetes Ingress 对外暴露服务](https://blog.csdn.net/qianghaohao/article/details/99354304)

[k8s ingress原理及ingress-nginx部署测试](https://segmentfault.com/a/1190000019908991)

