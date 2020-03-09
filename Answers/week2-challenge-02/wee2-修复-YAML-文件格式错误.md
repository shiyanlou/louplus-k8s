# 挑战：修复 YAML 文件格式错误

修改 `rss-site-deployment.yaml` 文件格式为如下所示：

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: rss-site
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        – name: front-end
          image: nginx
          ports:
            – containerPort: 80
        – name: rss-reader
          image: nickchase/rss-php-nginx:v1
          ports:
            – containerPort: 88
```
