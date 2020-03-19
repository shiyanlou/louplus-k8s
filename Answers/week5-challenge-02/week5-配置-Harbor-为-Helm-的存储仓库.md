# 挑战：配置 Harbor 为 Helm 的存储仓库

安装 Helm：

```bash
$ wget https://labfile.oss.aliyuncs.com/courses/1457/helm-v3.0.2-linux-amd64.tar.gz
$ tar -zxvf helm-v3.0.2-linux-amd64.tar.gz
$ sudo cp linux-amd64/helm /usr/local/bin/helm
# 安装成功，查看 Helm 的版本
$ helm version
version.BuildInfo{Version:"v3.0.2", GitCommit:"19e47ee3283ae98139d98460de796c1be1e3975f", GitTreeState:"clean", GoVersion:"go1.13.5"}
# 添加官方 Chart 仓库
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com
"stable" has been added to your repositories
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "stable" chart repository
Update Complete. Happy Helming!
```

使用 helm 安装 harbor：

```bash
# 添加仓库
$ helm repo add goharbor https://helm.goharbor.io
"goharbor" has been added to your repositories
$ helm repo ls
NAME        URL
stable      https://kubernetes-charts.storage.googleapis.com
goharbor    https://helm.goharbor.io
# 安装 harbor，关闭持久存储以及 tls 认证，设置服务访问方式为 NodePort，并且指定外部访问地址为 http://10.192.0.2:30002
$ helm install harbor goharbor/harbor --set persistence.enabled=false --set expose.tls.enabled=false --set expose.type=nodePort --set externalURL=http://10.192.0.2:30002
NAME: harbor
LAST DEPLOYED: Wed Mar  4 17:50:39 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Please wait for several minutes for Harbor deployment to complete.
Then you should be able to visit the Harbor portal at http://10.192.0.2:30002.
For more details, please visit https://github.com/goharbor/harbor.
```

然后在浏览器访问 `10.192.0.2:30002` 地址，结果如下所示，默认账号为 `admin`，密码为 `Harbor12345`：

![image](https://doc.shiyanlou.com/courses/1527/600404/91aebd5de30b0e9a4b490421939e5da2-0/wm)

新建一个仓库，命名为 `chart_repo`，访问级别为 `Public`：

![image](https://doc.shiyanlou.com/courses/1527/600404/bb6bc8504bd036e657a229ddc9925391-0/wm)

新建一个用户，用户名为 `shiyanlou`，邮箱为 `shiyanlou@simplecloud.com`，密码为 `Shiyanlou123`：

![image](https://doc.shiyanlou.com/courses/1527/600404/e9ef2e21af906aa5039bf81a5ee2b1ba-0/wm)

添加仓库地址，并配置推送：

```bash
# 添加仓库地址
$ helm repo add test_repo http://10.192.0.2:30002/chartrepo/chart_repo
"test_repo" has been added to your repositories
$ helm repo ls
NAME        URL
stable      https://kubernetes-charts.storage.googleapis.com
goharbor    https://helm.goharbor.io
test_repo   http://10.192.0.2:30002/chartrepo/chart_repo
# 安装 helm-push 插件
# 这里可能遇到 GitHub 访问超时严重的问题，可以逐个查找哪个 DNS 可以访问，然后配置到 /etc/hosts 中
$ helm plugin install https://github.com/chartmuseum/helm-push
Downloading and installing helm-push v0.8.1 ...
https://github.com/chartmuseum/helm-push/releases/download/v0.8.1/helm-push_0.8.1_linux_amd64.tar.gz
Installed plugin: push
# 查看插件
$ helm plugin ls
NAME    VERSION DESCRIPTION
push    0.8.1   Push chart package to ChartMuseum
# 获取一个 MySQL Chart 作为测试
$ helm fetch stable/mysql
$ ls
mysql-1.6.2.tgz
# 直接将打包好的 Chart 推送到远端仓库中，这里需要指定 Harbor 的用户名和密码
$ helm push mysql-1.6.2.tgz test_repo -u admin -p Harbor12345
Pushing mysql-1.6.2.tgz to test_repo...
Done.
```

![images](https://doc.shiyanlou.com/courses/1527/600404/74682b181e7615f56d6b0f21b29c0bf2-0/wm)
