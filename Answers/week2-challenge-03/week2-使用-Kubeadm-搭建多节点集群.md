# 挑战：使用 Kubeadm 搭建多节点集群

1.记录 Node 节点的公网、私网 IP 地址。

在阿里云 ECS 上开启 3 个按量付费的实例，我这里创建的 3 个实例的名称、公网 IP、以及私有 IP 分别如下所示（大家新建的公网 IP 和私有 IP 会不相同的）：

```text
节点名          公网 IP          私有 IP
kube-master    121.40.210.220  172.16.98.144
kube-node-1    121.40.214.218  172.16.98.145
kube-node-2    121.40.227.17   172.16.98.146
```

2.在 3 个节点上依次执行《阿里云 ECS 安装 Kubeadm》教程中 “安装 ALL-In-One 的 Kubernetes 集群” 部分的 1-4 步骤，即：安装 Docker，配置内核模块和参数，安装 kubeadm、kubelet、kubectl、ipvsadm 和 ipset，配置 Docker。

需要注意的是如果 kubeadm、kubelet、kubectl 工具的版本太新，可能对应的资源在阿里云源中没有，所以可以安装指定版本号的工具：

```bash
apt-get remove -y kubelet kubeadm kubectl
apt-get install -y  kubeadm=1.16.0-00 kubectl=1.16.0-00 kubelet=1.16.0-00
```

3.修改节点名。

设置所有节点主机名：

```bash
# 在 kube-master 节点执行
hostnamectl --static set-hostname  kube-master
reboot   # 修改节点名后需要重启才能生效

# 在 kube-node-1 节点执行
hostnamectl --static set-hostname  kube-node-1
reboot   # 修改节点名后需要重启才能生效

# 在 kube-node-2 节点执行
hostnamectl --static set-hostname  kube-node-2
reboot   # 修改节点名后需要重启才能生效
```

所有节点的 IP/主机名 加入 hosts 解析。编辑所有节点的 /etc/hosts 文件，都加入以下内容：（注意：将文件中类似于 `172.16.98.144 iZbp187zqzhr7zxedv9l37Z iZbp187zqzhr7zxedv9l37Z` 部分注释掉）

```bash
172.16.98.144 kube-master
172.16.98.145 kube-node-1
172.16.98.146 kube-node-2
```

4.初始化 kube-master 集群，复制 kubectl 相关配置文件。

在 Master 节点进行集群初始化，使用的命令为：

```bash
kubeadm init --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16
```

注意记录初始化成功后输出的 `kubeadm join` 命令，这个命令用于后面 Node 节点加入集群，比如我这里的输出为：`kubeadm join 172.16.98.144:6443 --token d3ab9p.cyiqd0js34def0g4 \ --discovery-token-ca-cert-hash sha256:141a38eebcfb07e319e551e7a52a8768e791619d2ab416754a7949d8298aa502`。

复制 kubectl 相关配置文件：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

5.将 Node 节点加入集群，安装网络插件 flannel，验证多节点集群搭建成功。

在 kube-node-1 和 kube-node-2 节点分别执行上一步骤输出的 `kubeadm join` 命令：

```bash
$ kubeadm join 172.16.98.144:6443 --token d3ab9p.cyiqd0js34def0g4 \
>     --discovery-token-ca-cert-hash sha256:141a38eebcfb07e319e551e7a52a8768e791619d2ab416754a7949d8298aa502
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 19.03.2. Latest validated version: 18.09
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.15" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

安装 flannel 网络插件。

提前在 3 个节点都拉取镜像，并重新打标签：

```bash
$ docker pull registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/flannel:v0.12.0-amd64
$ docker tag registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/flannel:v0.12.0-amd64 quay.io/coreos/flannel:v0.12.0-amd64
```

在 kube-master 节点下载 yaml 文件，并执行创建：

```bash
$ wget https://labfile.oss.aliyuncs.com/courses/1494/kube-flannel.yml
$ kubectl create -f kube-flannel.yml
```

检查 Node 是否成功加入集群。在 kube-master 节点执行如下命令：

```bash
$ kubectl get pods -n kube-system
NAME                                  READY   STATUS    RESTARTS   AGE
coredns-58cc8c89f4-8qxx7              1/1     Running   0          34m
coredns-58cc8c89f4-rj5f4              1/1     Running   0          34m
etcd-kube-master                      1/1     Running   0          33m
kube-apiserver-kube-master            1/1     Running   0          33m
kube-controller-manager-kube-master   1/1     Running   0          34m
kube-flannel-ds-amd64-2zpcp           1/1     Running   0          33m
kube-flannel-ds-amd64-dcqqn           1/1     Running   0          22m
kube-flannel-ds-amd64-jmh95           1/1     Running   0          22m
kube-proxy-2rp85                      1/1     Running   0          22m
kube-proxy-kt5jf                      1/1     Running   0          22m
kube-proxy-rzzkx                      1/1     Running   0          34m
kube-scheduler-kube-master            1/1     Running   0          33m
$ kubectl get nodes
NAME          STATUS   ROLES    AGE   VERSION
kube-master   Ready    master   35m   v1.16.0
kube-node-1   Ready    <none>   23m   v1.16.0
kube-node-2   Ready    <none>   23m   v1.16.0
```

7.验证集群。

在 kube-master 节点新建 `nginx.yaml` 文件，并执行如下命令创建资源：

```bash
$ kubectl create -f nginx.yaml
deployment.apps/nginx created
service/nginx-svc created
```

查看创建的 nginx-svc 和 Pod 的分布：

```bash
$ kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   51m
nginx-svc    ClusterIP   10.98.105.132   <none>        80/TCP    4s

$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
nginx-b96c8b7fb-62ssl   1/1     Running   0          17s   10.244.1.3   kube-node-1   <none>           <none>
nginx-b96c8b7fb-qrzsz   1/1     Running   0          17s   10.244.1.4   kube-node-1   <none>           <none>
nginx-b96c8b7fb-snwf4   1/1     Running   0          17s   10.244.2.3   kube-node-2   <none>           <none>
```

可以看到有两个 Pod 分布在 kube-node-1 节点，一个 Pod 分布在 kube-node-2 节点。

通过 svc 访问 nginx 服务，在 kube-master 节点执行如下命令：

```bash
# 这里是 ClusterIP:Port
$ curl 10.98.105.132:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

最后删除部署的资源：

```bash
$ kubectl delete -f nginx.yaml
deployment.apps "nginx" deleted
service "nginx-svc" deleted
```
