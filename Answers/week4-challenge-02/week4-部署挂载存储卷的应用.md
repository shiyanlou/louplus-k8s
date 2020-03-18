# 挑战：部署挂载存储卷的应用

使用 Kubeadm 在阿里云 ECS 上创建一个单节点 Kubernetes 集群的步骤这里就省略了。

这里用户为 `root`，主目录为 `/root`。

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
Export list for iZbp1ikd8ja49b9u9hm8auZ:
/root/nfs *
```

在共享文件目录下新创建一个名为 abc 的文件：

```bash
cd /root/nfs
touch abc
```

现在开始创建 PV、PVC、Pod。

```bash
cd
```

首先创建 PV，在 `/root` 目录下新建 `pv.yaml` 文件，并向其中写入如下内容：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mypv # pv 的名称为 mypv
spec:
  capacity:
    storage: 5Gi # 存储空间大小为 5GiB
  accessModes:
    - ReadWriteOnce # 读写权限，只能被单个 Node 挂载
  persistentVolumeReclaimPolicy: Recycle # # 回收策略，设置为回收空间
  nfs: # 存储卷类型为 nfs
    path: /root/nfs # 路径为 /root/nfs
    server: localhost # 服务器地址为 localhost，因为就是在本地创建
```

执行创建 PV：

```bash
$ kubectl create -f pv.yaml
persistentvolume/mypv created
$ kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM             STORAGECLASS   REASON   AGE
mypv   5Gi        RWO            Recycle          Available                                         11s
```

然后创建 PVC，在 `/root` 目录下新建 `pvc.yaml` 文件，并向其中写入如下内容：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim # pvc 的名称为 myclaim
spec:
  accessModes:
    - ReadWriteOnce # 读写权限，只能被单个 Node 挂载
  resources:
    requests:
      storage: 1Gi # 需要存储空间大小为 1GiB
  storageClassName: "" # 设置 StorageClass 为空
```

执行创建 PVC：

```bash
$ kubectl create -f pvc.yaml
persistentvolumeclaim/myclaim created
$ kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
mypv   5Gi        RWO            Recycle          Bound    default/myclaim                           4m24s
$ kubectl get pvc
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myclaim   Bound    mypv     5Gi        RWO                           5m56s
```

最后创建 Pod，在 `/root` 目录下新建 `pod.yaml` 文件，并向其中写入如下内容：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod # Pod 的名称为 mypod
spec:
  containers:
    - name: mycontainer # 容器名称为 mycontainer
      image: registry.cn-hangzhou.aliyuncs.com/louplus-linux/nginx:1.9.1 # 镜像为 nginx
      ports:
        - containerPort: 80 # 设置端口号为 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/var/data" # 外部卷挂载到容器中的 /var/data 目录下
          name: data # 卷名称为 data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: myclaim # 绑定名为 myclaim 的 pvc
        readOnly: true # 只具有可读权限
```

执行创建 Pod：

```bash
$ kubectl create -f pod.yaml
pod/mypod created
$ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
mypod   1/1     Running   0          2m44s
```

查看是否挂载成功：

```bash
$ kubectl exec -it mypod /bin/bash
root@mypod:/# ls /var/data
abc
root@mypod:/# exit
exit
# 查看磁盘的一个使用情况
$ df -h|grep localhost
localhost:/root/nfs   40G  3.5G   34G  10% /var/lib/kubelet/pods/d20058ce-6e22-4b82-892a-752617c275b3/volumes/kubernetes.io~nfs/mypv
```
