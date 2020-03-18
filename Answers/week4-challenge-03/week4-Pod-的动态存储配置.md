# 挑战：Pod 的动态存储配置

使用 Kubeadm 在阿里云 ECS 上创建一个单节点 Kubernetes 集群的步骤这里就省略了。

#### 安装配置 NFS 服务端

首先安装 NFS 服务端：

```bash
apt install -y nfs-kernel-server
```

创建共享目录，这里设置共享目录为 `/root/nfs`：

```bash
mkdir /root/nfs
```

配置以 root 权限共享 `/root/nfs` 目录：

```bash
$ vim /etc/exports
# 向其中添加如下内容：
/root/nfs    *(rw,sync,no_root_squash,no_subtree_check)
# rw：可读写权限，sync：资料同步写入到内存和磁盘中，no_root_squash：权限不压缩, 远程客户端拥有root权限，no_subtree_check：共享子目录时, 不检查父目录的权限
```

重启：

```bash
systemctl restart nfs-kernel-server
```

查看共享文件情况：

```bash
$ showmount -e localhost
Export list for iZbp15hy2fqh7xdgl5ezw9Z:
/root/nfs *
```

#### 安装配置 Kubernetes NFS-Client Provisioner 插件

由于 NFS 不支持自动创建 PV，可以利用社区提供的插件 nfs-client-provisioner Pod 来完成 PV 的自动创建，也就是说这个插件实现了 StorageClass 自动创建 PV 的需求。当 NFS 创建完成后，再被其它的 Pod 进行引用。

由于 nfs-client-provisioner 插件需要访问 apiServer，使用 RBAC 授权策略。在 `/root` 目录下新建 `rbac.yaml` 文件，并向其中写入如下内容：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

执行创建：

```bash
$ kubectl create -f rbac.yaml
serviceaccount/nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
```

然后创建 nfs-client-provisioner 插件（其实是一个 Pod），在 `/root` 目录下新建 `nfs-deployment.yaml` 文件，并向其中写入如下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
spec:
  selector:
    matchLabels:
      app: nfs-client-provisioner
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: localhost # nfs 服务器的地址
            - name: NFS_PATH
              value: /root/nfs # 共享文件的路径，这里可以根据实际情况填写你们设置的共享文件路径
      volumes:
        - name: nfs-client-root
          nfs:
            server: localhost # nfs 服务器的地址
            path: /root/nfs # 共享文件的路径，这里可以根据实际情况填写你们设置的共享文件路径
```

执行创建：

```bash
$ kubectl create -f nfs-deployment.yaml
deployment.apps/nfs-client-provisioner created
```

最后创建 StorageClass，在 `/root` 目录下新建 `class.yaml` 文件，并向其中写入如下内容：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: default # 设置 StorageClass 的名称为 default
provisioner: fuseim.pri/ifs  # 这里填写的是 provisioner 的名称，需要与上面 deployment 环境变量 PROVISIONER_NAME 的值一致
```

执行创建：

```bash
$ kubectl create -f class.yaml
storageclass.storage.k8s.io/default created
$ kubectl get sc
NAME      PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
default   fuseim.pri/ifs   Delete          Immediate           false                  17s
```

设置名为 default 的 StorageClass 为 Kubernetes 集群中的默认存储后端：

```bash
$ kubectl patch storageclass default -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/default patched
$ kubectl get sc
NAME                PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
default (default)   fuseim.pri/ifs   Delete          Immediate           false                  2m47s
```

#### 进行测试

好了，这样就完成了配置，现在进行测试：

在 `/root` 目录下新建 `test-claim.yaml` 文件，并向其中写入如下内容：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-claim
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

执行创建：

```bash
$ kubectl create -f test-claim.yaml
persistentvolumeclaim/test-claim created
$ kubectl get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-claim   Bound    pvc-0809a9d6-fbb3-4652-b675-6910457a179c   1Mi        RWX            default        103s
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   REASON   AGE
pvc-0809a9d6-fbb3-4652-b675-6910457a179c   1Mi        RWX            Delete           Bound    default/test-claim   default                 4m10s
```

启动一个测试 Pod，用于在名为 test-claim 的 PVC 中新创建一个 SUCCESS 文件。在 `/root` 目录下新建 `test-pod.yaml` 文件，并向其中写入如下内容：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-pod
      image: busybox:1.24
      command:
        - "/bin/sh"
      args:
        - "-c"
        - "touch /mnt/SUCCESS && exit 0 || exit 1"
      volumeMounts:
        - name: nfs-pvc
          mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim
```

执行创建：

```bash
$ kubectl create -f test-pod.yaml
pod/test-pod created
$ kubectl get pod
NAME                                      READY   STATUS      RESTARTS   AGE
nfs-client-provisioner-758c459469-8j6pj   1/1     Running     0          9m54s
test-pod                                  0/1     Completed   0          40s
```

当 test-pod 的状态变为 Completed，就说明该 Pod 已经执行完毕。现在我们去查看主机上是否存在刚刚创建的 SUCCESS 文件：

```bash
$ cd /root/nfs
$ ls
default-test-claim-pvc-0809a9d6-fbb3-4652-b675-6910457a179c
$ cd default-test-claim-pvc-0809a9d6-fbb3-4652-b675-6910457a179c
$ ls
SUCCESS
```

存在这个文件，这样就说明我们的部署没有问题，并且可以动态分配和使用 NFS 共享存储卷。
