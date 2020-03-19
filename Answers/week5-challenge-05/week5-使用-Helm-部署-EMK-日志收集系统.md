# 挑战：使用 Helm 部署 EMK 日志收集系统

一、搭建基础环境

由于使用在线环境会导致特别的卡顿，所以这里依然是在阿里云上使用 kubeadm 搭建的三节点集群，其中一个节点为 master，另外两个为 node。具体的搭建过程大家可以参考第二周课程中的《阿里云 ECS 安装 Kubeadm》实验。

这里简单罗列一下 3 个节点的 IP 地址：（注意这里将每个节点都进行了命名）

```text
节点名          公网IP           私有IP
kube-master    101.37.148.9    172.16.98.180
kube-node-1    47.97.81.80     172.16.98.181
kube-node-2    47.97.5.13      172.16.98.182
```

搭建成功的节点如下所示：

```bash
$ kubectl get nodes
NAME          STATUS   ROLES    AGE     VERSION
kube-master   Ready    master   5m20s   v1.17.0
kube-node-1   Ready    <none>   3m16s   v1.17.0
kube-node-2   Ready    <none>   3m10s   v1.17.0
```

下载安装 helm，并且设置本地存储卷：

```bash
# 下载 helm v2 的压缩包
$ wget https://labfile.oss.aliyuncs.com/courses/1502/helm-v2.16.1-linux-amd64.tar.gz
$ tar -zxvf helm-v2.16.1-darwin-amd64.tar.gz
$ cd linux-amd64
$ cp helm /usr/local/bin
$ cp tiller /usr/local/bin
# 下载 tiller 的配置文件，主要是创建 ServiceAccount 以及创建集群角色绑定
$ wget https://labfile.oss.aliyuncs.com/courses/1502/rbac-config.yaml
$ kubectl create -f rbac-config.yaml
serviceaccount/tiller created
clusterrolebinding.rbac.authorization.k8s.io/tiller created
# 初始化，指定使用国内阿里云镜像
$ helm init --service-account tiller -i registry.aliyuncs.com/google_containers/tiller:v2.16.1
# 查看 tiller Pod 是否创建成功
$ kubectl get pods -n kube-system|grep tiller
tiller-deploy-59f87b6744-w9xf5              1/1     Running            0          9m52s
# 查看 helm 版本信息，必须能够获取到 Client 和 Server 的信息才算安装成功
$ helm version
Client: &version.Version{SemVer:"v2.16.1", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.16.1", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"}

# 下载本地存储卷配置文件，主要是创建了 local-storage StorageClass，并且在每个节点上绑定了分别名为 local-pv-1 和 local-pv-2 的 PV。
$ wget https://labfile.oss.aliyuncs.com/courses/1502/local-pv.yaml
$ kubectl create -f local-pv.yaml
storageclass.storage.k8s.io/local-storage created
persistentvolume/local-pv-1 created
persistentvolume/local-pv-2 created
# 查看 StorageClass
$ kubectl get storageclass
NAME            PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-storage   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  26s
# 查看 PV
$ kubectl get pv
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    REASON   AGE
local-pv-1   30Gi       RWO            Retain           Available           local-storage            38s
local-pv-2   30Gi       RWO            Retain           Available           local-storage            38s
```

二、使用 Helm 安装 Elasticsearch：

```bash
# 添加 elastic 官方仓库
$ helm repo add elastic https://helm.elastic.co
"elastic" has been added to your repositories
# 查找是否有 elasticsearch 对应的 helm 包
$ helm search elasticsearch
NAME                         	CHART VERSION	APP VERSION	DESCRIPTION
elastic/elasticsearch        	7.5.1        	7.5.1      	Official Elastic helm chart for Elasticsearch
stable/elasticsearch         	1.32.1       	6.8.2      	Flexible and powerful open source, distributed real-time ...
...
# 由于社区提供的安装文件中的 image 在国内拉取不下来，我们再修改一些字段值以符合使用需求。
$ cd ~/.helm/cache/archive
# 获取对应的 elasticsearch 文件
$ helm fetch elastic/elasticsearch
$ ls
elasticsearch-7.5.1.tgz
# 解压缩
$ tar -zxvf elasticsearch-7.5.1.tgz
$ vim elasticsearch/values.yaml
# 将 replicas 的值修改为 2，将 image 的值修改为 registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/elasticsearch，修改 imageTag 为 7.5.1，在 volumeClaimTemplate 中添加一项 storageClassName: local-storage。
# 修改完成，进行打包
$ helm package elasticsearch
Successfully packaged chart and saved it to: /home/shiyanlou/.helm/cache/archive/elasticsearch-7.5.1.tgz
# 使用修改后的文件进行安装，并命名为 elasticsearch
$ helm install --name elasticsearch elasticsearch-7.5.1.tgz
NAME:   elasticsearch
LAST DEPLOYED: Tue Jan 14 17:53:35 2020
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                    AGE
elasticsearch-master-0  0s
elasticsearch-master-1  0s
elasticsearch-master-2  0s

==> v1/Service
NAME                           AGE
elasticsearch-master           1s
elasticsearch-master-headless  1s

==> v1/StatefulSet
NAME                  AGE
elasticsearch-master  1s

==> v1beta1/PodDisruptionBudget
NAME                      AGE
elasticsearch-master-pdb  1s


NOTES:
1. Watch all cluster members come up.
  $ kubectl get pods --namespace=default -l app=elasticsearch-master -w
2. Test cluster health using Helm test.
  $ helm test elasticsearch

# 可以查看 elasticsearch Pod 的创建过程
$ kubectl get pods --namespace=default -l app=elasticsearch-master -w
NAME                     READY   STATUS     RESTARTS   AGE
elasticsearch-master-0   0/1     Init:0/1   0          20s
elasticsearch-master-1   0/1     Init:0/1   0          20s
elasticsearch-master-0   0/1     PodInitializing   0          31s
elasticsearch-master-1   0/1     PodInitializing   0          31s
elasticsearch-master-0   0/1     Running           0          32s
elasticsearch-master-1   0/1     Running           0          32s
elasticsearch-master-0   1/1     Running           0          88s
elasticsearch-master-1   1/1     Running           0          89s

$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
elasticsearch-master-0   1/1     Running   0          3m34s
elasticsearch-master-1   1/1     Running   0          3m34s
# 使用端口转发的方式在本地尝试访问 elasticsearch
$ kubectl port-forward svc/elasticsearch-master 9200
Forwarding from 127.0.0.1:9200 -> 9200
# 访问 elasticsearch，出现相关信息，则说明安装成功
$ curl localhost:9200
{
  "name" : "elasticsearch-master-0",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "zSeKjhPzSUu34lWIfgSnHg",
  "version" : {
    "number" : "7.5.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "3ae9ac9a93c95bd0cdc054951cf95d88e1e18d96",
    "build_date" : "2019-12-16T22:57:37.835892Z",
    "build_snapshot" : false,
    "lucene_version" : "8.3.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

三、使用 Helm 安装 Kibana：

```bash
$ cd ~/.helm/cache/archive
$ helm fetch elastic/kibana
$ tar -zxvf kibana-7.5.1.tgz
$ vim kibana/values.yaml
# 修改 image 为 registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/kibana，修改 imageTag 为 7.5.1
$ helm package kibana
Successfully packaged chart and saved it to: /Users/yuanchunrong/.helm/cache/archive/kibana-7.5.1.tgz
# 使用修改后的文件进行安装，并命名为 kibana
$ helm install --name kibana kibana-7.5.1.tgz
NAME:   kibana
LAST DEPLOYED: Tue Jan 14 16:08:25 2020
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME           AGE
kibana-kibana  0s

==> v1/Pod(related)
NAME                            AGE
kibana-kibana-74f594d749-fpb2l  0s

==> v1/Service
NAME           AGE
kibana-kibana  0s
# 安装完成后查看 kibana Pod，如果 describe kibana Pod 可能会查看 warn:readiness 错误的警告，这里暂时可以不用管
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
elasticsearch-master-0           1/1     Running   0          38m
elasticsearch-master-1           1/1     Running   0          38m
kibana-kibana-6cb4db8b88-fshws   1/1     Running   0          2m7s
```

四、使用 Helm 安装 Metricbeat

```bash
$ cd ~/.helm/cache/archive
$ helm fetch elastic/metricbeat
$ tar -zxvf metricbeat-7.5.1.tgz
$ vim metricbeat/values.yaml
# 修改 image 为 registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/metricbeat，修改 imageTag 为 7.5.1
$ helm package metricbeat
Successfully packaged chart and saved it to: /root/.helm/cache/archive/metricbeat-7.5.1.tgz
# 使用修改后的文件进行安装，并命名为 metricbeat
$ helm install --name metricbeat metricbeat-7.5.1.tgz
NAME:   metricbeat
LAST DEPLOYED: Wed Jan 15 16:26:57 2020
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                          AGE
metricbeat-metricbeat-config  1s

==> v1/DaemonSet
NAME                   AGE
metricbeat-metricbeat  1s

==> v1/Deployment
NAME                           AGE
metricbeat-kube-state-metrics  1s
metricbeat-metricbeat-metrics  1s

==> v1/Pod(related)
NAME                                            AGE
metricbeat-kube-state-metrics-78775f5d68-f29dp  0s
metricbeat-metricbeat-84z2m                     0s
metricbeat-metricbeat-hxwcf                     1s
metricbeat-metricbeat-metrics-5b96bd4f7d-szm2g  0s

==> v1/Service
NAME                           AGE
metricbeat-kube-state-metrics  1s

==> v1/ServiceAccount
NAME                           AGE
metricbeat-kube-state-metrics  1s
metricbeat-metricbeat          1s

==> v1beta1/ClusterRole
NAME                                AGE
metricbeat-kube-state-metrics       1s
metricbeat-metricbeat-cluster-role  1s

==> v1beta1/ClusterRoleBinding
NAME                                        AGE
metricbeat-kube-state-metrics               1s
metricbeat-metricbeat-cluster-role-binding  1s


NOTES:
1. Watch all containers come up.
  $ kubectl get pods --namespace=default -l app=metricbeat-metricbeat -w

# 查看 metricbeat 相关的 Pod
$ kubectl get pods
NAME                                             READY   STATUS    RESTARTS   AGE
...
metricbeat-kube-state-metrics-78775f5d68-f29dp   1/1     Running   0          2m29s
metricbeat-metricbeat-84z2m                      1/1     Running   0          2m29s
metricbeat-metricbeat-hxwcf                      1/1     Running   0          2m30s
metricbeat-metricbeat-metrics-5b96bd4f7d-szm2g   1/1     Running   0          2m29s
# 在前面对 elasticsearch 进行了本地 9200 端口转发，可以看到对应的指标已经在 elasticsearch 中建立了索引
$ curl localhost:9200/_cat/indices
green open .kibana_task_manager_1             MwpKnOz5QJCYGBy3FzqONA 1 1    2 1 32.5kb 16.2kb
green open .apm-agent-configuration           rLdmps8-Tme7O5G5OeGcBg 1 1    0 0   566b   283b
green open metricbeat-7.5.1-2020.01.15-000001 HHriSR2TSUGqpCpD4cR2uQ 1 1 1404 0    2mb  996kb
green open .kibana_1                          dFQoCyYiSkOTNshe1dyclg 1 1    2 0 18.7kb  9.3kb
```

五、在浏览器访问 Kibana

在集群中 kibana 只有一个 ClusterIP，可以通过 NodePort 的方式暴露服务：

```bash
$ kubectl expose deployment/kibana-kibana --type=NodePort --name=kibana-nodeport
service/kibana-nodeport exposed
# 查看 kibana-nodeport 服务映射的端口
$ kubectl get svc
NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
...
kibana-kibana                   ClusterIP   10.96.9.95      <none>        5601/TCP            33m
kibana-nodeport                 NodePort    10.96.45.51     <none>        5601:31054/TCP      5s
...
```

登录阿里云，添加集群所在节点的安全组访问规则，这里添加端口 `31054/31054`，现在就可以在集群外部直接访问啦。

在浏览器访问 `http://<Master 公网IP>:30433`，在这里就是 `http://101.37.148.9:31054/`，会自动跳转到 Kibana 主页：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20200115-1579082789687/wm)

依次点击 `Management -> Kibana -> Index Patterns -> Create index pattern`：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20200115-1579082816087/wm)

在 `Define index pattern` 中填入 `mtricbeat-*`：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20200115-1579082996408/wm)

在 `Configure settings` 中选择 `@timestamp`：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20200115-1579082826407/wm)

可以看到 `mtricbeat-*` 详细字段：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20200115-1579082853577/wm)

最后点击 `Discover` 可以看到由 Metricbeat 收集到的集群指标展示如下：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20200115-1579082862347/wm)
