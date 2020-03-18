# 挑战：Kubectl 多集群访问配置

1. 在阿里云 ECS 上开启 3 个按量付费的实例，我这里创建的 3 个实例的名称、公网 IP、以及私有 IP 分别如下所示（大家新建的公网 IP 和私有 IP 会不相同的）：

```text
节点名        公网IP           私有IP
work-manage  116.62.104.68   172.16.98.183
work-dev     101.37.91.224   172.16.98.184
work-prod    118.178.187.158 172.16.98.185
```

2. 修改节点名

设置所有节点主机名：

```bash
# 在 work-manage 节点执行
hostnamectl --static set-hostname  work-manage
# 在 work-dev 节点执行
hostnamectl --static set-hostname  work-dev
# 在 work-prod 节点执行
hostnamectl --static set-hostname  work-prod
# 修改完主机名以后，3 个节点都需要执行重启
reboot
```

3. 参考第二周课程中的《阿里云 ECS 安装 Kubeadm》实验进行配置，这里只列出比较重要的步骤

需要注意的是如果 kubeadm、kubelet、kubectl 工具的版本太新，可能对应的资源在阿里云源中没有，所以可以安装指定版本号的工具：

```bash
apt-get remove -y kubelet kubeadm kubectl
apt-get install -y  kubeadm=1.16.0-00 kubectl=1.16.0-00 kubelet=1.16.0-00
```

使用 kubeadm 初始化集群（在 3 个节点都执行如下命令）：

```bash
kubeadm init --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16
```

复制 kubectl 相关配置文件（在 3 个节点都执行如下命令）：

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

安装网络插件 flannel（在 3 个节点都执行如下命令）：

```bash
docker pull quay.io/coreos/flannel:v0.12.0-amd64
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

查看节点是否部署成功：

```bash
# 其余两个节点的命令类似
root@work-manage:~# kubectl get nodes
NAME        STATUS   ROLES    AGE     VERSION
work-manage   Ready    master   5m10s   v1.17.0
```

4. 在 work-manage 集群中执行如下命令

```bash
# 进入配置目录
$ cd .kube
# 先将 work-manage 集群中的 config 进行备份
$ cp config config-work-manage
# 将 work-dev 集群中的配置文件拷贝到本地来，输入密码即可
$ scp root@101.37.91.224:/etc/kubernetes/admin.conf ~/.kube/config-work-dev
# 将 work-prod 集群中的配置文件拷贝到本地来，输入密码即可
$ scp root@118.178.187.158:/etc/kubernetes/admin.conf ~/.kube/config-work-prod
# 查看所有的配置文件
$ ls
cache  config  config-work-dev  config-work-prod  config-work-manage  http-cache
```

需要规范配置文件的命名，主要是 config-work-dev 和 config-work-prod 文件，需要修改集群名称、用户名称、和上下文关联名称：

```yaml
# config-work-dev
...
# 修改集群名称
- cluster:
    server: https://172.16.98.184:6443
  name: work-dev-cluster
# 更新上下文名称，关联对应用户以及集群
contexts:
- context:
    cluster: work-dev-cluster
    user: work-dev-admin
  name: work-dev
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
# 修改用户名称
users:
- name: work-dev-admin
  user:
```

```yaml
# config-work-prod
...
# 修改集群名称
- cluster:
    server: https://172.16.98.185:6443
  name: work-prod-cluster
# 更新上下文名称，关联对应用户以及集群
contexts:
- context:
    cluster: work-prod-cluster
    user: work-prod-admin
  name: work-prod
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
# 修改用户名称
users:
- name: work-prod-admin
  user:
```

使用 KUBECONFIG 环境变量保存配置文件路径列表：

```bash
cd
vim .bashrc
# 写入如下内容
export KUBECONFIG=$HOME/.kube/config-work-manage:$HOME/.kube/config-work-dev:$HOME/.kube/config-work-prod

source .bashrc
```

检查效果：

```bash
# 查看环境变量
$ echo $KUBECONFIG
/root/.kube/config-work-manage:/root/.kube/config-work-dev:/root/.kube/config-work-prod
# 查看所有的集群上下文列表
$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER             AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes          kubernetes-admin
          work-dev                      work-dev-cluster    work-dev-admin
          work-prod                     work-prod-cluster   work-prod-admin
# 查看当前上下文
$ kubectl config current-context
kubernetes-admin@kubernetes
# 现在将上下文切换为 work-dev 开发环境
$ kubectl config use-context work-dev
Switched to context "work-dev".
# 验证是否已经切换到 work-dev 开发环境，通过 nodes 名称知道已经切换成功
$ kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
work-dev   Ready    master   40m   v1.17.0
```

查看所有的配置信息：

```bash
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.16.98.183:6443
  name: kubernetes
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.16.98.184:6443
  name: work-dev-cluster
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.16.98.185:6443
  name: work-prod-cluster
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
- context:
    cluster: work-dev-cluster
    user: work-dev-admin
  name: work-dev
- context:
    cluster: work-prod-cluster
    user: work-prod-admin
  name: work-prod
current-context: work-prod
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: work-dev-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: work-prod-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```
