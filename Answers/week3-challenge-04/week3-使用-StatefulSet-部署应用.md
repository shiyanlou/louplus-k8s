# 挑战：使用 StatefulSet 部署应用

在 /home/shiyanlou 目录下新建 `local-storage.yaml` 文件，并向其中写入如下代码：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

执行创建：

```bash
$ kubectl create -f local-storage.yaml
storageclass.storage.k8s.io/local-storage created
```

在 `/home/shiyanlou` 目录下新建 `local-pv.yaml` 文件，并向其中写入如下内容：

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /tmp
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - kube-node-1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv2
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /tmp
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - kube-node-2
```

执行创建：

```bash
$ kubectl create -f local-pv.yaml
persistentvolume/local-pv1 created
persistentvolume/local-pv2 created
```

在 `/home/shiyanlou` 目录下新建 `mongo.yaml` 文件，并向其中写入如下代码：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
  labels:
    app: mongo
spec:
  ports:
    - port: 27017
      targetPort: 27017
      name: mongo
  clusterIP: None
  selector:
    app: mongo
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 2
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongo
          image: registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/mongo:latest
          command:
            - mongod
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-persistent-storage
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongo-persistent-storage
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: local-storage
        resources:
          requests:
            storage: 100Mi
```

在执行创建之前可以监听 Pod 的创建过程，在一个新的终端执行如下命令：

```bash
kubectl get pods -w -l app=mongo
```

然后执行创建：

```bash
$ kubectl create -f mongo.yaml
service/mongo created
statefulset.apps/mongo created
```

查看创建过程：

```bash
$ kubectl get pods -w -l app=mongo
NAME      READY   STATUS              RESTARTS   AGE
mongo-0   0/1     Pending             0          0s
mongo-0   0/1     Pending             0          0s
mongo-0   0/1     ContainerCreating   0          18s
mongo-0   1/1     Running             0          35s
mongo-1   0/1     Pending             0          0s
mongo-1   0/1     Pending             0          1s
mongo-1   0/1     ContainerCreating   0          1s
mongo-1   1/1     Running             0          31s
```

查看创建成功的 Pod：

```bash
$ kubectl get pods -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
mongo-0   1/1     Running   0          69s   10.244.2.4   kube-node-1   <none>           <none>
mongo-1   1/1     Running   0          34s   10.244.3.3   kube-node-2   <none>           <none>
```

检查 PVC：

```bash
$ kubectl get pvc
NAME                               STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS    AGE
mongo-persistent-storage-mongo-0   Bound    local-pv1   1Gi        RWO            local-storage   3m2s
mongo-persistent-storage-mongo-1   Bound    local-pv2   1Gi        RWO            local-storage   2m27s
```
