# 挑战：集群添加 shixiaolou 账户并设置 RBAC 授权

一、启动 Minikube 集群并查看 Config 配置信息。

```bash
# 启动本地的 Minikube 集群
$ minikube start --vm-driver=virtualbox --registry-mirror=https://docker.mirrors.ustc.edu.cn --kubernetes-version v1.15.2
# 先查看 Minikube 集群的配置：有一个 cluster 名为 minikube，当前上下文为 minikube，有一个 user 名为 minikube
$ cat ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /Users/xxx/.minikube/ca.crt
    server: https://192.168.99.101:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /Users/xxx/.minikube/client.crt
    client-key: /Users/xxx/.minikube/client.key
```

二、为 shixiaolou 账户生成私钥（key）和认证签名请求文件（csr），然后向集群提交申请，审批通过后获得证书（crt），使用私钥和证书向 context 中注册账户。

```bash
# 创建一个名为 resource 的文件夹，并进入该文件夹，后续的文件内容都保存在该文件夹下
$ mkdir resource && cd resource
# 使用 openssl genrsa 命令生成私钥，名为 shixiaolou.key
$ openssl genrsa -out shixiaolou.key 2048
Generating RSA private key, 2048 bit long modulus
.....................................................................................................................+++
..........................+++
e is 65537 (0x10001)
# openssl req 命令使用上面的 shixiaolou.key 私钥生成一个认证签名请求文件，名为 shixiaolou.csr。
$ openssl req -new -key shixiaolou.key -out shixiaolou.csr -subj "/CN=shixiaolou/O=shiyanlou"\n
$ ls
shixiaolou.csr  shixiaolou.key
# 使用 base64 对认证签名进行加密
$ cat shixiaolou.csr | base64 | tr -d '\n'
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ2FEQ0NBVkFDQVFBd0l6RU9NQXdHQTFVRUF3d0ZRMmhwYm1FeEVUQVBCZ05WQkFvTUNHTm9hVzVsYzJWdQpNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQTF1YnBMOEJXVFpXa3VWdnk1U1hjCm9INmZRNHl3dEs5WFhOZ1hYcXlDL25aejdGb1ZxTDl0Smk4eFFHbk15QnlqcWxwUHBzb0lWNks1Q2NWTG12SXoKYURjeUNJcTU2UUR1N0Zsb29pdmVCWkhTMGY3SURCeVZZdHJGUzAveHpkTzRBaFp1TnAyNjFvL0Q0N2RFLytyawpFL0VGbXVxQnY5eW9lTGhuTzBmY3RhQTdNbEZoek4xa1VBRXQwR250dnBGdzRzNEtwd0dDK2JtOHVzTTdMZUZZCmRyeDRWeEtPc2hITWtBRW51K2RjMTlwL2dUNlpMaVMyUlREc2dValBhSytKeWxEb2tnSXRhYWIwdEJVVVlxOG4KQjNDMTRBdFQreFV5cmZNak5nYS80WnJkQzJVS1FLTWdLOThyUjZaLytGV0xycFdsSVlTOC9sK0RHaVpoemhwcwpmd0lEQVFBQm9BQXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBS3hwLzZ1YUxVdm1vWXVnWjFSL2poMTFrSmkyClBUTDFTZzVwY1NlU0c2VEVjbkgzVFdONDBJRW42eUdWNEhGMWszUlJWNjZlUU1nWkxMUEM2UHIwS1d6U01PaVYKR2ZkcmFHWjhiUTVoQW5RVUpXMU9CU0U4TVNpc09idFQ3MXZnOFI1OWltajRJL1RXcXQ3bGljNkh5bHQyQnN0WQpmV2owQmtGbS9IZUYzc0RGMzhnYlNSdGlzS0NLWnNUSEE0a1VMd3dTemR2RVVGbWhKWFpiTXRqa2Z5bTJJVmxPCkJKZXJoUEcvdlZoWldLaDF2NHZEZmkzeXZSZCtuTXdxT0dXL3EveUsrb01tUnJHS2hZTi80Mmdnc09yaHNzRE0KVzNZZXl3eGxSMWdwOFp6QzhMSW5wWDVoS1JPQ1ZvY1RHb1A0RXNQWXBZS2tvMWx2MlVJWDdueXZ6aDg9Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
# 使用 Kubernetes Certificate Signing Request 资源对象创建 YAML 文件，将加密后的认证签名放入其中，然后将证书签名请求提交给 kubernetes 集群。这样就可以将私钥和集群关联起来。
$ touch signing-request.yaml

apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: shixiaolou-csr  # 将这个请求命名为 shixiaolou-csr
spec:
  groups:
  - system:authenticated # 分配为系统认证组

  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ2FEQ0NBVkFDQVFBd0l6RU9NQXdHQTFVRUF3d0ZRMmhwYm1FeEVUQVBCZ05WQkFvTUNHTm9hVzVsYzJWdQpNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQTF1YnBMOEJXVFpXa3VWdnk1U1hjCm9INmZRNHl3dEs5WFhOZ1hYcXlDL25aejdGb1ZxTDl0Smk4eFFHbk15QnlqcWxwUHBzb0lWNks1Q2NWTG12SXoKYURjeUNJcTU2UUR1N0Zsb29pdmVCWkhTMGY3SURCeVZZdHJGUzAveHpkTzRBaFp1TnAyNjFvL0Q0N2RFLytyawpFL0VGbXVxQnY5eW9lTGhuTzBmY3RhQTdNbEZoek4xa1VBRXQwR250dnBGdzRzNEtwd0dDK2JtOHVzTTdMZUZZCmRyeDRWeEtPc2hITWtBRW51K2RjMTlwL2dUNlpMaVMyUlREc2dValBhSytKeWxEb2tnSXRhYWIwdEJVVVlxOG4KQjNDMTRBdFQreFV5cmZNak5nYS80WnJkQzJVS1FLTWdLOThyUjZaLytGV0xycFdsSVlTOC9sK0RHaVpoemhwcwpmd0lEQVFBQm9BQXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBS3hwLzZ1YUxVdm1vWXVnWjFSL2poMTFrSmkyClBUTDFTZzVwY1NlU0c2VEVjbkgzVFdONDBJRW42eUdWNEhGMWszUlJWNjZlUU1nWkxMUEM2UHIwS1d6U01PaVYKR2ZkcmFHWjhiUTVoQW5RVUpXMU9CU0U4TVNpc09idFQ3MXZnOFI1OWltajRJL1RXcXQ3bGljNkh5bHQyQnN0WQpmV2owQmtGbS9IZUYzc0RGMzhnYlNSdGlzS0NLWnNUSEE0a1VMd3dTemR2RVVGbWhKWFpiTXRqa2Z5bTJJVmxPCkJKZXJoUEcvdlZoWldLaDF2NHZEZmkzeXZSZCtuTXdxT0dXL3EveUsrb01tUnJHS2hZTi80Mmdnc09yaHNzRE0KVzNZZXl3eGxSMWdwOFp6QzhMSW5wWDVoS1JPQ1ZvY1RHb1A0RXNQWXBZS2tvMWx2MlVJWDdueXZ6aDg9Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=

  usages:
  - digital signature
  - key encipherment
  - server auth

# 执行创建
$ kubectl create -f signing-request.yaml
certificatesigningrequest.certificates.k8s.io/shixiaolou-csr created
# 查看 csr 的状态，可以看到请求是 Pending 状态。
$ kubectl get csr
NAME             AGE   REQUESTOR       CONDITION
shixiaolou-csr   23s   minikube-user   Pending
# 集群管理员使用 approve 命令批准请求，使其生效。
$ kubectl certificate approve shixiaolou-csr
certificatesigningrequest.certificates.k8s.io/shixiaolou-csr approved
# shixiaolou-csr 资源变为 Approved,Issued 状态
$ kubectl get csr
NAME             AGE   REQUESTOR       CONDITION
shixiaolou-csr   62s   minikube-user   Approved,Issued
# 证书已经被批准颁发，从集群中获取签名信息，并使用 base64 解密，最后保存为 shixiaolou.crt。
$ kubectl get csr shixiaolou-csr -o jsonpath='{.status.certificate}' | base64 --decode > shixiaolou.crt
# shixiaolou.crt 是集群批准的验证身份的客户端证书，shixiaolou.key 是私钥。通过这两个文件就可以配置 shixiaolou 账户的身份验证。
# 为 kubeconfig 配置文件添加 shixiaolou 用户。
$ kubectl config set-credentials shixiaolou --client-certificate=shixiaolou.crt --client-key=shixiaolou.key
User "shixiaolou" set.
# 查看配置是否生效
$ cat ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /Users/xxx/.minikube/ca.crt
    server: https://192.168.99.101:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /Users/xxx/.minikube/client.crt
    client-key: /Users/xxx/.minikube/client.key
- name: shixiaolou  # 成功添加了 shixiaolou 账户
  user:
    client-certificate: /Users/xxx/Desktop/cred/shixiaolou.crt
    client-key: /Users/xxx/Desktop/cred/shixiaolou.key
```

三、创建 sxl-test 命名空间。

```bash
# 创建 sxl-test 命名空间
$ kubectl create ns sxl-test
namespace/sxl-test created
# 查看命名空间
$ kubectl get ns
NAME              STATUS   AGE
default           Active   200d
kube-node-lease   Active   200d
kube-public       Active   200d
kube-system       Active   200d
sxl-test          Active   10s
# kubectl auth 命令可以验证特定用户的权限，先查看当前管理员账户在 sxl-test 空间下是否可以查看 pod
$ kubectl auth can-i list pods --namespace sxl-test
yes
# 然后查看 shixiaolou 账户在 sxl-test 空间下是否有权限，可以看到是没有权限的
$ kubectl auth can-i list pods --namespace sxl-test --as shixiaolou
no
# 可以设置一个账户为 shixiaolou、集群使用 minikube、命名空间为 sxl-test 的上下文环境
$ kubectl config set-context test-context --cluster=minikube --namespace=sxl-test --user=shixiaolou
Context "test-context" created.
# 查看设置好的上下文环境
$ cat ~/.kube/config
...
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
- context:
    cluster: minikube
    namespace: sxl-test
    user: shixiaolou
  name: test-context
current-context: minikube
...
```

四、在 sxl-test 命名空间下，创建一个简单的 busybox Pod。然后在 sxl-test 命名空间下为 shixiaolou 账户创建角色绑定，让其拥有对于 pods、services、nodes 资源对象的 get、watch、list 操作权限。最后还需要为 shixiaolou 账户创建集群角色绑定，让其拥有对于 nodes 资源对象的 get、watch、list 操作权限。

```bash
# 在 sxl-test 命名空间下，创建一个简单的 busybox Pod
$ touch myapp.yaml

apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: sxl-test
  labels:
    app: myapp
spec:
  containers:
  - name: myapp
    image: busybox   # 在 minikube 中拉取镜像可能会比较慢，需要耐心等待
    command: ["/bin/sh", "-ec", "while :; do echo '.'; sleep 5 ; done"]

# 执行创建，默认使用集群管理员角色进行创建
$ kubectl create -f myapp.yaml
pod/myapp created
# 查看创建好的 Pod
$ kubectl get pods -n sxl-test
NAME    READY   STATUS    RESTARTS   AGE
myapp   1/1     Running   0          16m
# shixiaolou 账户在 sxl-test 命名空间下还没有权限查看 Pod
$ kubectl get pods -n sxl-test --as shixiaolou
Error from server (Forbidden): pods is forbidden: User "bob" cannot list resource "pods" in API group "" in the namespace "sxl-test"
# 给 shixiaolou 账户设置在 sxl-test 命名空间下的权限绑定
$ touch sxl-namespace-role.yaml

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: sxl-test
  name: sxl-test-reader
rules:
- apiGroups: [""] # "" 表示 core API 组
  resources: ["pods", "services", "nodes"] # 对于 pods、services、nodes 资源对象有 get、watch、list 的操作权限
  verbs: ["get", "watch", "list"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: sxl-test-read-access
  namespace: sxl-test
subjects:
- kind: User
  name: shixiaolou # 指明为 shixiaolou 账户
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role # 角色绑定
  name: sxl-test-reader # 匹配上面创建的 sxl-test-reader 角色
  apiGroup: rbac.authorization.k8s.io

# 执行创建
$ kubectl create -f namespace-role.yaml
role.rbac.authorization.k8s.io/sxl-test-reader created
rolebinding.rbac.authorization.k8s.io/sxl-test-read-access created
# 在 sxl-test 命名空间授权以后就有权限查看 pod 了
$ kubectl get pods -n sxl-test --as shixiaolou
NAME    READY   STATUS    RESTARTS   AGE
myapp   1/1     Running   0          37m

# 但是 shixiaolou 账户还没有访问整个集群级别资源的权限
$ kubectl get nodes --as shixiaolou
Error from server (Forbidden): nodes is forbidden: User "shixiaolou" cannot list resource "nodes" in API group "" at the cluster scope
# 现在创建一个 ClusterRole，并将其绑定到 shixiaolou 账户
$ touch sxl-cluster-role.yaml

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]  # 这里只赋予对于 nodes 资源对象拥有 get、watch、list 的操作权限
  verbs: ["get", "watch", "list"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-cluster-nodes
subjects:
- kind: User
  name: shixiaolou # 绑定到 shixiaolou 账户
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-node-reader
  apiGroup: rbac.authorization.k8s.io
# 执行创建
$ kubectl create -f cluster-role.yaml
clusterrole.rbac.authorization.k8s.io/cluster-node-reader created
clusterrolebinding.rbac.authorization.k8s.io/read-cluster-nodes created
# 现在 shixiaolou 账户就有查看整个集群 nodes 的权限
$ kubectl get nodes --as shixiaolou
NAME       STATUS   ROLES    AGE    VERSION
minikube   Ready    <none>   188d   v1.15.2

# 最后暂停整个集群
$ minikube stop
```
