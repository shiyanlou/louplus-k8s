# 挑战：使用 Helm 部署 Tomcat

安装 Helm：

```bash
$ wget https://labfile.oss.aliyuncs.com/courses/1457/helm-v3.0.2-linux-amd64.tar.gz
$ tar -zxvf helm-v3.0.2-linux-amd64.tar.gz
$ sudo cp linux-amd64/helm /usr/local/bin/helm
# 安装成功，查看 Helm 的版本
$ helm version
version.BuildInfo{Version:"v3.0.2", GitCommit:"19e47ee3283ae98139d98460de796c1be1e3975f", GitTreeState:"clean", GoVersion:"go1.13.5"}
# 添加一个 Chart 仓库，这里设置阿里云 Chart 镜像仓库为标准仓库
$ helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
"stable" has been added to your repositories
```

创建应用：

```bash
$ helm create tomcat
Creating tomcat
```

修改配置信息：

```bash
vim tomcat/templates/deployment.yaml
# 主要修改如下所示
      containers:
        - name: {{ .Chart.Name }}
          ...
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}" # 修改使用镜像标签
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8080 # 设置容器端口为 8080
              protocol: TCP
          #livenessProbe:   # 删除存活探针和就绪探针的检测
          #  httpGet:
          #    path: /
          #    port: http
          #readinessProbe:
          #  httpGet:
          #    path: /
          #    port: http

```

```bash
vim tomcat/values.yaml

image:
  repository: registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/tomcat-app   # 修改镜像仓库地址
  tag: v1      # 设置镜像版本号
  pullPolicy: IfNotPresent
...
service:
  type: NodePort   # 将服务修改为 NodePort 类型
  port: 80
```

修改完成后，执行测试和安装：

```bash
# 测试
$ helm install --debug --dry-run test tomcat/
# 安装
$ helm install tomcat tomcat/
NAME: tomcat
LAST DEPLOYED: Mon Mar  2 14:23:02 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services tomcat)
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
```

访问测试：

```bash
$ export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services tomcat)
$ export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
$ echo http://$NODE_IP:$NODE_PORT
http://10.192.0.2:31391
```

复制上面获取到的链接地址，然后在浏览器打开即可访问：

![](https://doc.shiyanlou.com/courses/1527/600404/ec49ce2b4ad8e448c80c291c4cf1b096-0/wm)
