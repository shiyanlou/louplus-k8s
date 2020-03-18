# 挑战：使用 ConfigMap 配置 Pod

由于实验环境为 dind，直接运行 MySQL Pod 会报错，暂时可以使用如下命令解决：

```bash
sudo ln -s /etc/apparmor.d/usr.sbin.mysqld /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.mysqld
```

从文件创建名为 mysql-config 的 ConfigMap：

```bash
$ kubectl create configmap mysql-config --from-file=mysqld.cnf
configmap/mysql-config created
```

使用 key-value 键值对创建名为 myconfig 的 ConfigMap：

```bash
$ kubectl create configmap myconfig --from-literal=v1=shiyanlou
configmap/myconfig created
```

补充完整的 `mysql-d.yaml` 文件内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - image: registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/mysql:5.7
          name: mysql
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                configMapKeyRef:
                  name: myconfig
                  key: v1
          volumeMounts:
            - name: mysql
              mountPath: /etc/mysql/mysql.conf.d
      volumes:
        - name: mysql
          configMap:
            name: mysql-config
```

执行创建：

```bash
$ kubectl create -f mysql-d.yaml
deployment.apps/mysql created
```

验证文件是否挂载成功，以及 MySQL 密码是否能够使用：

```bash
$ kubectl get pods
mysql-6574f97df5-wc2vg   1/1     Running     0          52s

$ kubectl exec -it mysql-6574f97df5-wc2vg bash
root@mysql-6574f97df5-wc2vg:/# mysql -uroot -pshiyanlou
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.27 MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> exit;
Bye
root@mysql-6574f97df5-wc2vg:~# cat /etc/mysql/mysql.conf.d/mysqld.cnf
[client]
port = 3306
socket = /var/run/mysqld/mysqld.sock
[mysql]
no-auto-rehash
[mysqld]
user = mysql
port = 3306
socket = /var/run/mysqld/mysqld.sock
datadir = /var/lib/mysql
[mysqld_safe]
log-error= /var/log/mysql/mysql_oldboy.err
pid-file = /var/run/mysqld/mysqld.pid
```

通过 Pod 的具体信息也可以查看 ConfigMap 的配置：

```bash
$ kubectl describe pod mysql-6574f97df5-wc2vg
......

    Environment:
      MYSQL_ROOT_PASSWORD:  <set to the key 'v1' of config map 'myconfig'>  Optional: false
    Mounts:
      /etc/mysql/mysql.conf.d from mysql (rw)

......
```

另外在这里介绍一下卷挂载的方式可以使用 ConfigMap 热更新，比如我们修改 mysql-config ConfigMap，在 [mysqld] 字段下添加一项配置 `server-id = 1`，等待一会儿进入容器查看就可以发现文件的配置也相应进行了更改（热更新需要的时间相对要长一点，可能在一到两分钟左右）：

```bash
$ kubectl edit configmap mysql-config
configmap/mysql-config edited

$ kubectl exec -it mysql-6574f97df5-wc2vg bash
root@mysql-6574f97df5-wc2vg:~# cat /etc/mysql/mysql.conf.d/mysqld.cnf
[client]
port = 3306
socket = /var/run/mysqld/mysqld.sock
[mysql]
no-auto-rehash
[mysqld]
user = mysql
port = 3306
socket = /var/run/mysqld/mysqld.sock
datadir = /var/lib/mysql
server-id = 1 # 完成了热更新，配置文件已经修改
[mysqld_safe]
log-error= /var/log/mysql/mysql_oldboy.err
pid-file = /var/run/mysqld/mysqld.pid
```
