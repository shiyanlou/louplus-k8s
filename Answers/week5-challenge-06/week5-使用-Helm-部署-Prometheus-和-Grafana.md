# 挑战：使用 Helm 部署 Prometheus 和 Grafana

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

新建一个 monitoring 目录，因为获取到 Chart 之后需要修改文件：

```bash
mkdir monitoring
cd monitoring
```

安装 prometheus：

```bash
$ helm fetch stable/prometheus
$ ls
prometheus-11.0.1.tgz
$ tar -zxvf prometheus-11.0.1.tgz
# 查看 prometheus/values.yaml 文件中的具体设置
$ cat prometheus/values.yaml
$ touch prometheus-values.yaml
# 这里为了方便，不设置持久存储
alertmanager:
  persistentVolume:
    enabled: false
server:
  persistentVolume:
    enabled: false

$ helm install prometheus stable/prometheus -f prometheus-values.yaml
NAME: prometheus
LAST DEPLOYED: Wed Mar  4 14:29:17 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.default.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace default port-forward $POD_NAME 9090
$ kubectl get pods
NAME                                             READY   STATUS    RESTARTS   AGE
prometheus-alertmanager-85f7d8555f-5449f         2/2     Running   0          2m
prometheus-kube-state-metrics-685dccc6d8-plsvf   1/1     Running   0          2m
prometheus-node-exporter-9rzpz                   1/1     Running   0          2m
prometheus-node-exporter-hf66k                   1/1     Running   0          2m
prometheus-pushgateway-55679fd67f-w8btb          1/1     Running   0          2m
prometheus-server-5fbc688c44-79m9k               1/2     Running   0          2m
# 设置端口转发
$ export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
$ kubectl --namespace default port-forward $POD_NAME 9090
Forwarding from 127.0.0.1:9090 -> 9090
```

然后在浏览器访问 `127.0.0.1:9090` 地址，可以看到正确的 Prometheus 页面：

![image](https://doc.shiyanlou.com/courses/1527/600404/3ccfd105353eb672a4c4771245401a4a-0/wm)

接下来安装 grafana：

```bash
$ helm fetch stable/grafana
$ ls
grafana-5.0.5.tgz
$ tar -zxvf grafana-5.0.5.tgz
# 查看配置，发现不需要设置存储，这里暂时就不再做修改
$ cat grafana/values.yaml
# 直接安装
$ helm install grafana stable/grafana
NAME: grafana
LAST DEPLOYED: Wed Mar  4 14:46:48 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   grafana.default.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:

     export POD_NAME=$(kubectl get pods --namespace default -l "app=grafana,release=grafana" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace default port-forward $POD_NAME 3000

3. Login with the password from step 1 and the username: admin

$ kubectl get pods
NAME                                             READY   STATUS    RESTARTS   AGE
grafana-678b6c5cc-jpqbt                          1/1     Running   0          2m20s
...
# 查看 grafana 的登录密码
$ kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
D4Wgk9TlkntC3vVXyNP61CySa39KKajY1UbYE8rn

$ kubectl --namespace default port-forward grafana-678b6c5cc-jpqbt 3000
Forwarding from 127.0.0.1:3000 -> 3000
```

在浏览器访问 `http://localhost:3000`，结果如下所示：

![image](https://doc.shiyanlou.com/courses/1527/600404/cb76ab31746df786c946dc29c38ad91f-0/wm)

登录成功以后点击主页面的添加数据 `Add data source`，然后选择 `Prometheus`，这里需要配置 Prometheus URL 地址，只需要填写 ClusterIP 即可，使用如下命令查看：

```bash
$ kubectl get svc|grep prometheus-server
prometheus-server               ClusterIP   10.99.49.21     <none>        80/TCP     43m
```

![image](https://doc.shiyanlou.com/courses/1527/600404/97ba8eb6e6f586c98e550f499c026976-0/wm)

最后点击页面最下方的 `Save & Test`。看到提示 `Data source is working` 表示配置成功。

我们可以先使用 Prometheus 默认提供的 Dashboards 先查看效果如何：

![image](https://doc.shiyanlou.com/courses/1527/600404/ad8e626e68d880f2707b7ee9b8f1b34b-0/wm)

![image](https://doc.shiyanlou.com/courses/1527/600404/6fabd897c691418678e6c8d1e8656d50-0/wm)

效果还可以。

这里还可以使用不同的模板 json 文件配置不同的监控项，这里提供了两个 json 文件。`node-monitoring.json` 主要用于监控模板，`pod-monitoring.json` 主要用于监控 Pod。使用如下命令下载：

```bash
wget https://labfile.oss.aliyuncs.com/courses/1527/node-monitoring.json
wget https://labfile.oss.aliyuncs.com/courses/1527/pod-monitoring.json
```

在 `Dashboards --> Manage` 中选择 `Import`，之后选择 `Upload .json file` 上传文件，可以自定义 Dashboard 的名称，注意数据源选择 `Prometheus`。

![image](https://doc.shiyanlou.com/courses/1527/600404/00a8a13d8b6b6e25c09ce052373d5a4d-0/wm)

看起来效果更好一些：

![image](https://doc.shiyanlou.com/courses/1527/600404/73ff9975d3aa50b270d38368cf250c72-0/wm)

![image](https://doc.shiyanlou.com/courses/1527/600404/1c80766d8e17f728fcfb9bc7d0952718-0/wm)
