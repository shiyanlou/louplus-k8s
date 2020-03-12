# 挑战：限制 Docker 用户权限

#### 挑战一：设置 Docker 的 uid 和 gid，避免文件被误删

```bash
# 设置用户的 uid 和 gid 为 1000
$ docker run --user=1000:1000 --rm -it alpine:latest id
uid=1000 gid=1000
# 对文件进行备份
$ sudo cp /bin/ls /bin/ls.bak && ls /bin/ls.bak
/bin/ls.bak
# 设置用户的 uid 和 gid 为 1000，查看文件是否还能被删除
$ docker run --user=1000:1000 --rm -it -v /bin:/host alpine:latest rm -f /host/ls.bak
rm: can't remove '/host/ls.bak': Permission denied
```

#### 挑战二：限制 Docker 用户获取更高的权限

```bash
# 使用 --security-opt=no-new-privileges 参数可以限制用户获取更高的权限
$ docker run -u 1000 --security-opt=no-new-privileges registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/test_priv:1.0
Effective uid: 1000
```