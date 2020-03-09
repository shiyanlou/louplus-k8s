# 挑战：配置 kube-proxy 组件

当 Kubernetes 集群运行的服务数量很多时，kube-proxy 工作在 ipvs 模式将获得更好的性能。默认情况下 kube-proxy 允许在 iptables 模式下运行，但我们可以通过修改 kube-proxy ConfigMap 快速将 kube-proxy 切换到 ipvs 模式。

首先，通过下面的命令更新 kube-proxy ConfigMap：

```bash
kubectl -n kube-system edit cm kube-proxy
```

修改 `mode` 字段值为 `ipvs`，如下图所示：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190831-1567215358006/wm)

重启 kube-proxy：

```bash
kubectl -n kube-system rollout restart ds/kube-proxy
```

当重启完成后可以通过如下命令验证 kube-proxy 已经允许在 ipvs 模式下运行：

```bash
kubectl -n kube-system logs -f -l k8s-app=kube-proxy --all-containers
```

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190831-1567215565561/wm)

同时通过命令 `ipvsadm -L` 也将会发现各种转发规则已经成功添加。
