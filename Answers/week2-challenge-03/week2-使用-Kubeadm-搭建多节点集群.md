# 挑战：使用 Kubeadm 搭建多节点集群

1. 在阿里云 ECS 上开启 3 个按量付费的实例，我这里创建的 3 个实例的名称、公网 IP、以及私有 IP 分别如下所示（大家新建的公网 IP 和私有 IP 会不相同的）：

```text
节点名          公网 IP          私有 IP
kube-master    121.40.210.220  172.16.98.144
kube-node-1    121.40.214.218  172.16.98.145
kube-node-2    121.40.227.17   172.16.98.146
```

2. 修改节点名

设置所有节点主机名：

```bash
# 在 kube-master 节点执行
hostnamectl --static set-hostname  kube-master
# 在 kube-node-1 节点执行
hostnamectl --static set-hostname  kube-node-1
# 在 kube-node-2 节点执行
hostnamectl --static set-hostname  kube-node-2
```

所有节点的 主机名/IP 加入 hosts 解析，编辑所有节点的 /etc/hosts 文件，都加入以下内容：（注意：将文件中类似于 `172.16.98.144 iZbp187zqzhr7zxedv9l37Z iZbp187zqzhr7zxedv9l37Z` 部分注释掉）

```bash
172.16.98.144 kube-master
172.16.98.145 kube-node-1
172.16.98.146 kube-node-2
```

3. 在 3 个节点上依次执行《Kubeadm 安装》教程中 “2.2 安装 ALL-In-One 的 Kubernetes 集群” 下面的 1-4 步骤

4. 初始化 kube-master 集群

对 kube-master 执行《Kubeadm 安装》教程中 “2.2 安装 ALL-In-One 的 Kubernetes 集群” 下面的 5-8 步骤，注意记录下第 5 步初始化成功后输出的 `kubeadm join` 命令，这个命令用于后面其它节点加入集群，比如我这里的输出为：`kubeadm join 172.16.98.144:6443 --token d3ab9p.cyiqd0js34def0g4 \ --discovery-token-ca-cert-hash sha256:141a38eebcfb07e319e551e7a52a8768e791619d2ab416754a7949d8298aa502`

这一步最终达到的效果如下所示：

```bash
$ kubectl get nodes
NAME          STATUS   ROLES    AGE     VERSION
kube-master   Ready    master   4m21s   v1.15.3
```

5. 将 Node 节点加入集群

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

6. 检查 Node 是否成功加入集群

在 kube-master 节点执行如下命令：

```bash
$ kubectl get nodes
NAME          STATUS   ROLES    AGE    VERSION
kube-master   Ready    master   9m8s   v1.15.3
kube-node-1   Ready    <none>   101s   v1.15.3
kube-node-2   Ready    <none>   66s    v1.15.3
```

7. 验证集群

在 kube-master 节点新建 `tomcat.yaml` 文件，并执行如下命令创建资源：

```bash
$ kubectl create -f tomcat.yaml
deployment.apps/tomcat created
service/tomcat-svc created
```

查看创建的 tomcat-svc 和 Pod 的分布：

```bash
$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    16m
tomcat-svc   ClusterIP   10.96.141.228   <none>        8080/TCP   26s

$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
tomcat-b96c8b7fb-62ssl   1/1     Running   0          17s   10.244.1.3   kube-node-1   <none>           <none>
tomcat-b96c8b7fb-qrzsz   1/1     Running   0          17s   10.244.1.4   kube-node-1   <none>           <none>
tomcat-b96c8b7fb-snwf4   1/1     Running   0          17s   10.244.2.3   kube-node-2   <none>           <none>
```

可以看到有两个 Pod 分布在 kube-node-1 节点，一个 Pod 分布在 kube-node-2 节点。

通过 svc 访问 tomcat 服务，在 kube-master 节点执行如下命令：

```bash
$ curl 10.96.141.228:8080

<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8" />
        <title>Apache Tomcat/8.5.45</title>
        <link href="favicon.ico" rel="icon" type="image/x-icon" />
        <link href="favicon.ico" rel="shortcut icon" type="image/x-icon" />
        <link href="tomcat.css" rel="stylesheet" type="text/css" />
    </head>
......
```

最后删除部署的资源：

```bash
$ kubectl delete -f tomcat.yaml
deployment.apps "tomcat" deleted
service "tomcat-svc" deleted
```
