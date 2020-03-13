# 挑战：修复 YAML 文件格式错误

修改 `rss-site-deployment.yaml` 文件格式为如下所示：

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rss-site
spec:
  selector:
    matchLabels:
      app: web
  replicas: 2
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: front-end
          image: registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/nginx:1.9.1
          ports:
            - containerPort: 80
        - name: rss-reader
          image: registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/rss-php-nginx:v1
          ports:
            - containerPort: 88
```
