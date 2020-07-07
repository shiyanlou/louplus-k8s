# 在集群中部署运行 VSCode

#### 安装 traefik

```bash
# 下载 traefik 资源文件夹
$ wget https://labfile.oss.aliyuncs.com/courses/1494/traefik.zip
# 解压缩
$ unzip traefik.zip
# 进入目录
$ cd traefik
# 查看文件列表
$ ls
crd.yaml  dashboard.yaml  deployment.yaml  rbac.yaml
# 执行创建资源对象
$ kubectl create -f .
customresourcedefinition.apiextensions.k8s.io/ingressroutes.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/ingressroutetcps.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/middlewares.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/tlsoptions.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/traefikservices.traefik.containo.us created
ingressroute.traefik.containo.us/traefik-dashboard created
deployment.apps/traefik created
clusterrole.rbac.authorization.k8s.io/traefik created
serviceaccount/traefik created
clusterrolebinding.rbac.authorization.k8s.io/traefik created
# 查看 traefik pods
$ kubectl get pods -n kube-system -l app=traefik -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
traefik-76467c46b-crjcf   1/1     Running   0          70s   10.192.0.4   kube-node-2   <none>           <none>
traefik-76467c46b-wbdnc   1/1     Running   0          70s   10.192.0.3   kube-node-1   <none>           <none>
# 查看 dashboard ingressroute
$ kubectl get ingressroute
NAME                AGE
traefik-dashboard   8m59s
```

#### 部署资源对象

将 vscode 部署在 default 命名空间下，使用 Deployment 资源对象创建对应的 Pod，使用的镜像为：`registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/code-server:latest`，通过环境变量 `PASSWORD` 设置访问 IDE 的密码为 `shiyanlou`。

```bash
$ mkdir vscode
$ cd vscode
```

新建一个 `vscode/vscode.yaml` 文件，并向其中写入如下内容：

```yaml
apiVersion: v1
kind: Service
metadata:
 name: vscode-service
spec:
 ports:
 - port: 80
   targetPort: 8080
 selector:
   app: vscode
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: vscode
  name: vscode
spec:
  selector:
    matchLabels:
      app: vscode
  template:
    metadata:
      labels:
        app: vscode
    spec:
      containers:
      - image: registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/code-server:latest
        imagePullPolicy: IfNotPresent
        name: vscode
        ports:
        - containerPort: 8080
        env:
        - name: PASSWORD
          value: "shiyanlou"
```

执行创建：

```bash
$ kubectl create -f vscode.yaml
service/vscode created
deployment.apps/vscode created
$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
vscode-7d5cb54cf6-6lfrc   1/1     Running   0          33s
```

我们这里要求使用 HTTPS 来访问应用，所以需要先创建一个自签名的证书：

```bash
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=code.shiyanlou.com"
# 通过 secret 对象引用证书文件
$ kubectl create secret tls vscode-tls --cert=tls.crt --key=tls.key
secret/vscode-tls created
```

新建一个 `vscode/ingress-route.yaml` 文件，并向其中写入如下内容：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
spec:
  redirectScheme:
    scheme: https
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: vscode
spec:
  entryPoints:
  - web
  routes:
  - kind: Rule
    match: Host(`code.shiyanlou.com`)
    middlewares:
    - name: redirect-https
    services:
    - kind: Service
      name: vscode-service
      port: 80
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: vscode-service-https
spec:
  entryPoints:
  - websecure
  routes:
  - kind: Rule
    match: Host(`code.shiyanlou.com`)
    services:
    - kind: Service
      name: vscode-service
      port: 80
  tls:
    secretName: vscode-tls  # 配置 tls 时使用前面创建的 vscode-tls secret
    domains:
    - main: '*.shiyanlou.com'
```

执行创建：

```bash
$ kubectl create -f ingress-route.yaml
middleware.traefik.containo.us/redirect-https created
ingressroute.traefik.containo.us/vscode created
ingressroute.traefik.containo.us/vscode-service-https created
# 查看 ingressroute
$ kubectl get ingressroute
NAME                   AGE
traefik-dashboard      10m
vscode                 10s
vscode-service-https   10s
```

在本地添加域名映射：

```bash
sudo vim /etc/hosts
# 添加映射

10.192.0.3      code.shiyanlou.com
```

在浏览器访问地址 `code.shiyanlou.com`，会跳转到 HTTPS 连接，然后进入登录页面，首先需要填写密码，这里的密码是 YAML 文件中设置的环境变量 `shiyanlou`：

由于采用的是自签名证书，这里有一个危险提示，不过不要紧：

![im](https://doc.shiyanlou.com/courses/1494/600404/a4bc8f1049c2b7110c0f38808f6ad359-0/wm)

登录页面需要密码：

![im](https://doc.shiyanlou.com/courses/1494/600404/23281d0f9348b3bb8b1cffa847eebe3e-0/wm)

成功登录，现在就可以在浏览器中直接使用 vscode 了：

![im](https://doc.shiyanlou.com/courses/1494/600404/9095b11746c52312b12dbf3ae61d53c8-0/wm)
