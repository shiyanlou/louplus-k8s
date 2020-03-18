# 挑战：设置 NodePort Service

在 `/home/shiyanlou` 目录下新建 `tomcat-deployment.yaml`，并向其中写入如下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat
spec:
  selector:
    matchLabels:
      app: tomcat
  replicas: 2
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
        - name: tomcat
          image: registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/tomcat-app:v1
          ports:
            - containerPort: 8080
```

执行创建：

```bash
$ kubectl create -f tomcat-deployment.yaml
deployment.apps/tomcat created
```

在 `/home/shiyanlou` 目录下新建 `tomcat-svc.yaml` 文件，并向其中写入如下内容：

```bash
apiVersion: v1
kind: Service
metadata:
  name: tomcat-svc
spec:
  type: NodePort
  ports:
  - port: 80 # 设置 ClusterIP 对应的端口为 80
    targetPort: 8080 # Pod 开放的端口为 8080
    nodePort: 30001 # 设置在 Node 上开放的端口为 30001
  selector:
    app: tomcat
```

执行创建：

```bash
$ kubectl create -f tomcat-svc.yaml
service/tomcat-svc created
# 创建 NodePort 类型的服务成功
$ kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
tomcat-svc   NodePort    10.98.46.27   <none>        80:30001/TCP   22m
```

可以先试着访问一下：

```bash
$ curl 10.192.0.2:30001
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8" />
        <title>Apache Tomcat/8.5.47</title>
        <link href="favicon.ico" rel="icon" type="image/x-icon" />
        <link href="favicon.ico" rel="shortcut icon" type="image/x-icon" />
        <link href="tomcat.css" rel="stylesheet" type="text/css" />
    </head>
...
```
