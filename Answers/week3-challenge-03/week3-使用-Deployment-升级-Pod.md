# 挑战：使用 Deployment 升级 Pod

由于实验环境为 dind，直接运行 MySQL Pod 会报错，暂时可以使用如下命令解决：

```bash
sudo ln -s /etc/apparmor.d/usr.sbin.mysqld /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.mysqld
```

在 `/home/shiyanlou` 目录下新建 `mysql-deployment.yaml` 文件，并写入如下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  selector:
    matchLabels:
      app: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: database
          image: registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/mysql:5.7
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "123456"
```

执行创建：

```bash
$ kubectl create -f mysql-deployment.yaml --record
deployment.apps/mysql-deployment created
```

查看 Deployment、RS、Pods：

```bash
$ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
mysql-deployment   3/3     3            3           9s

$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
mysql-deployment-5897447f6c   3         3         3       14s

$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
mysql-deployment-5897447f6c-26l49   1/1     Running   0          20s
mysql-deployment-5897447f6c-jllkd   1/1     Running   0          20s
mysql-deployment-5897447f6c-vh46l   1/1     Running   0          20s
```

执行升级，将 mysql:5.7 的镜像升级为 mysql:8.0 的镜像：

```bash
$ kubectl set image deployment/mysql-deployment database=registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/mysql:8.0
deployment.extensions/mysql-deployment image updated
```
