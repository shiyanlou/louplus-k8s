# 挑战：部署实验楼网站

`shiyanlou-rc.yaml` 文件内容如下：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: shiyanlou
spec:
  replicas: 1
  selector:
    app: shiyanlou-app
  template:
    metadata:
      labels:
        app: shiyanlou-app
    spec:
      containers:
        - name: shiyanlou
          image: registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/shiyanlou:1.0
          ports:
            - containerPort: 8080
```

`shiyanlou-svc.yaml` 文件内容如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shiyanlou
spec:
  type: NodePort
  ports:
    - port: 8080
      nodePort: 30001
  selector:
    app: shiyanlou-app
```
