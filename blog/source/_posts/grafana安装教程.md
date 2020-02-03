---
title: grafana安装教程
date: 2020-01-22 12:36:41
tags:
categories:
	- Kubernetes
---


### 配置文件

grafana-deploy.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: grafana
  namespace: kube-ops
  labels:
    app: grafana
spec:
  revisionHistoryLimit: 10
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:6.5.3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          name: grafana
        env:
        - name: GF_SECURITY_ADMIN_USER
          value: admin
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: admin321
        readinessProbe:
          failureThreshold: 10
          httpGet:
            path: /api/health
            port: 3000
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 30
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /api/health
            port: 3000
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: 100m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 256Mi
        volumeMounts:
        - mountPath: /var/lib/grafana
          subPath: grafana
          name: storage
      securityContext:
        fsGroup: 472
        runAsUser: 472
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: grafana
```

我们使用了最新的镜像grafana/grafana:6.5.3，然后添加了监控检查、资源声明，另外两个比较重要的环境变量GF_SECURITY_ADMIN_USER和GF_SECURITY_ADMIN_PASSWORD，用来配置 grafana 的管理员用户和密码的，由于 grafana 将 dashboard、插件这些数据保存在/var/lib/grafana这个目录下面的，所以我们这里如果需要做数据持久化的话，就需要针对这个目录进行 volume 挂载声明，其他的和我们之前的 Deployment 没什么区别，由于上面我们刚刚提到的 Changelog 中 grafana 的 userid 和 groupid 有所变化，所以我们这里需要增加一个securityContext的声明来进行声明。

当然如果要使用一个 pvc 对象来持久化数据，我们就需要添加一个可用的 pv 供 pvc 绑定使用：（grafana-volume.yaml）

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: grafana
spec:
  capacity:
    storage: 1Gi
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
  name: grafana
  namespace: kube-ops
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

```
如果不适用nfs，可以直接指定一台宿主机，然后设置为hostPath


```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: grafana
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  hostPath:
    path: /cache1/grafana/data/
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana
  namespace: kube-ops
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

```

最后，我们需要对外暴露 grafana 这个服务，所以我们需要一个对应的 Service 对象，当然用 NodePort 或者再建立一个 ingress 对象都是可行的：(grafana-svc.yaml)

```yaml

apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: kube-ops
  labels:
    app: grafana
spec:
  type: NodePort
  ports:
    - port: 3000
      nodePode: 30000
  selector:
    app: grafana
```
可以通过30000端口直接访问


现在我们直接创建上面的这些资源对象：

```
$ kubectl create -f grafana-volume.yaml
persistentvolume "grafana" created
persistentvolumeclaim "grafana" created
$ kubectl create -f grafana-deploy.yaml
deployment.extensions "grafana" created
$ kubectl create -f grafana-svc.yaml
service "grafana" created
```

创建完成后，我们可以查看 grafana 对应的 Pod 是否正常：
```
$ kubectl get pods -n kube-ops
NAME                          READY     STATUS             RESTARTS   AGE
grafana-5f7b965b55-wxvvk      0/1       CrashLoopBackOff   1          22s
```
我们可以看到这里的状态是CrashLoopBackOff，并没有正常启动，我们查看下这个 Pod 的日志：
```
$ kubectl logs -f grafana-5f7b965b55-wxvvk -n kube-ops
GF_PATHS_DATA='/var/lib/grafana' is not writable.
You may have issues with file permissions, more information here: http://docs.grafana.org/installation/docker/#migration-from-a-previous-version-of-the-docker-container-to-5-1-or-later
mkdir: cannot create directory '/var/lib/grafana/plugins': Permission denied
```
上面的错误是在5.1版本之后才会出现的，当然你也可以使用之前的版本来规避这个问题。

可以看到是日志中错误很明显就是/var/lib/grafana目录的权限问题，这还是因为5.1版本后 groupid 更改了引起的问题，我们这里增加了securityContext，但是我们将目录/var/lib/grafana挂载到 pvc 这边后目录的拥有者并不是上面的 grafana(472)这个用户了，所以我们需要更改下这个目录的所属用户，这个时候我们可以利用一个 Job 任务去更改下该目录的所属用户：（grafana-chown-job.yaml）

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: grafana-chown
  namespace: kube-ops
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: grafana-chown
        command: ["chown", "-R", "472:472", "/var/lib/grafana"]
        image: busybox
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: storage
          subPath: grafana
          mountPath: /var/lib/grafana
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: grafana
```

上面我们利用一个 busybox 镜像将/var/lib/grafana目录更改成了472这个 user 和 group，不过还需要注意的是下面的 volumeMounts 和 volumes 需要和上面的 Deployment 对应上。

现在我们删除之前创建的 Deployment 对象，重新创建：

```
$ kubectl delete -f grafana-deploy.yaml
deployment.extensions "grafana" deleted
$ kubectl create -f grafana-deploy.yaml
deployment.extensions "grafana" created
$ kubectl create -f grafana-chown-job.yaml
job.batch "grafana-chown" created
```
我们可以查看 Service 对象：
```
$ kubectl get svc -n kube-ops
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                          AGE
grafana      NodePort    10.97.46.27     <none>        3000:30105/TCP                   1h

```

现在我们就可以在浏览器中使用http://<任意节点IP:30000>来访问 grafana 这个服务了.

![image](https://note.youdao.com/yws/api/personal/file/1696F72387D64E5B96D39C8D7EAED328?method=download&shareKey=e95b41f49ecb0fc9dcfa3f5025b01b5b)


### Reference
https://www.qikqiak.com/k8s-book/docs/56.Grafana%E7%9A%84%E5%AE%89%E8%A3%85%E4%BD%BF%E7%94%A8.html