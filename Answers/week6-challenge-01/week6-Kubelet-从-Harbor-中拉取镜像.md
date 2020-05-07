# Kubelet-从-Harbor-中拉取镜像

在 harbor 中新建一个 shiyanlou 用户，这里将用户名和密码都设置为 shiyanlou。

然后使用命令行登录，在 shiyanlou 用户下 push 一个 nginx 镜像：

```bash
docker login registry.shiyanlou.com  # 这里登录用户名和密码都填写为 shiyanlou
docker pull nginx:1.9.1
docker tag nginx:1.9.1 registry.shiyanlou.com/library/nginx:1.9.1
docker push registry.shiyanlou.com/library/nginx:1.9.1
```

首先使用用户名和密码生成一个字符串，格式为 `用户名:密码`，然后使用 base64 进行加密。

```bash
$ echo "shiyanlou:shiyanlou" | base64
c2hpeWFubG91OnNoaXlhbmxvdQo=
```

新建一个名为 `dockerconfig.json` 的文件，填写上相关的信息：

```json
{
    "auths" : {
        "registry.shiyanlou.com" : {
            "auth" : "c2hpeWFubG91OnNoaXlhbmxvdQo=",
            "email" : "simplecloud@qq.com"
        }
    }
}
```

然后将整个 `dockerconfig.json` 文件使用 base64 加密：

```bash
$ cat dockerconfig.json | base64
ewogICAgImF1dGhzIiA6IHsKICAgICAgICAicmVnaXN0cnkuc2hpeWFubG91LmNvbSIgOiB7CiAgICAgICAgICAgICJhdXRoIiA6ICJjMmhwZVdGdWJHOTFPbk5vYVhsaGJteHZkUW89IiwKICAgICAgICAgICAgImVtYWlsIiA6ICJzaW1wbGVjbG91ZEBxcS5jb20iCiAgICAgICAgfQogICAgfQp9Cg==
```

在 `default` 命名空间下创建名为 `shiyanlou-secret` 的 secret。新建 `secret.yaml` 文件并向其中写入如下内容：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: shiyanlou-secret
  namespace: default
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: ewogICAgImF1dGhzIiA6IHsKICAgICAgICAicmVnaXN0cnkuc2hpeWFubG91LmNvbSIgOiB7CiAgICAgICAgICAgICJhdXRoIiA6ICJjMmhwZVdGdWJHOTFPbk5vYVhsaGJteHZkUW89IiwKICAgICAgICAgICAgImVtYWlsIiA6ICJzaW1wbGVjbG91ZEBxcS5jb20iCiAgICAgICAgfQogICAgfQp9Cg==
```

然后执行创建：

```bash
kubectl create -f secret.yaml
```

在部署应用时，为 Pod 指定下载镜像所需的 secret 名称，新建 `nginx.yaml` 文件并向其中写入如下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: registry.shiyanlou.com/library/nginx:1.9.1
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: shiyanlou-secret
```

执行创建：

```bash
kubectl create -f nginx.yaml
kubectl get pods
```

如果 nginx Pod 成功运行了，就说明已经成功从 Harbor 私有仓库中拉取到镜像啦。
