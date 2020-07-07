# 部署高可用 Etcd 集群

在阿里云上创建 4 个按量付费抢占式实例，使用 ubuntu16.04 系统，4 个节点的 IP 地址分别如下：

| 主机名 | 内网 IP 地址 |
| ----- |  ----------- |
| host1 | 172.16.99.25 |
| host2 | 172.16.99.26 |
| host3 | 172.16.99.27 |
| kube-master | 172.16.99.28 |

1. 下载和分发 etcd 二进制文件

在 host1 节点执行如下命令：

```bash
curl -L https://labfile.oss.aliyuncs.com/courses/1494/etcd-v3.4.9-linux-amd64.tar.gz -o etcd-v3.4.9-linux-amd64.tar.gz
tar -xzf etcd-v3.4.9-linux-amd64.tar.gz
mv etcd-v3.4.9-linux-amd64/etcd etcd-v3.4.9-linux-amd64/etcdctl /root
rm -rf etcd-v3.4.9-linux-amd64 etcd-v3.4.9-linux-amd64.tar.gz

./etcd --version
./etcdctl version
```

分发二进制文件到集群所有节点，新建 `distribution.sh` 文件并向其中写入如下内容：

```bash
CONTROL_PLANE_IPS="172.16.99.26 172.16.99.27"
for host in ${CONTROL_PLANE_IPS}; do
    echo ">>> $host"
    scp /root/etcd* root@$host:
    ssh root@$host "chmod +x *"
done
```

执行分发：

```bash
bash distribution.sh
```

2. 下载 cfssl 工具

在 host1 节点执行如下命令：

```bash
mkdir -p /usr/local/cfssl/bin
cd /usr/local/cfssl/bin
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O cfssl
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O cfssl-certinfo
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O cfssljson
chmod +x cfssl cfssl-certinfo cfssljson
echo 'export PATH=$PATH:/usr/local/cfssl/bin' >>/etc/bashrc
```

3. CA 配置文件

在 host1 节点执行如下命令。

首先生成 CA 配置文件，使用 profiles 指定不同的过期时间、使用场景等参数。

```bash
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "etcd": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",    # client 可以用该 CA 对 server 提供的证书进行验证
            "client auth"     # server 可以用该 CA 对 client 提供的证书进行验证
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```

接下来生成 CA 证书签名请求：

```bash
cat > ca-csr.json <<EOF
{
  "CN": "etcd",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Chengdu",
      "L": "Chengdu",
      "O": "etcd",
      "OU": "System"
    }
  ]
}
EOF
```

最后生成 CA 证书与密钥：

```bash
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
$ ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```

4. 创建 etcd TLS 证书与私钥

etcdctl 与 etcd 集群，以及 etcd 集群之间的通信都采用的是 TLS 加密。

在 host1 节点执行如下命令：

```bash
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  # hosts 字段指定授权使用证书的 etcd 节点 IP 地址。大家需要自行替换为自己的 IP 地址。
  "hosts": [
    "127.0.0.1",
    "172.16.99.25",
    "172.16.99.26",
    "172.16.99.27"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Chengdu",
      "L": "Chengdu",
      "O": "etcd",
      "OU": "System"
    }
  ]
}
EOF

# 生成 etcd 证书与私钥
cfssl gencert -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -profile=etcd etcd-csr.json | cfssljson -bare etcd
```

分发生成的证书和私钥到各 etcd 节点，新建 `dist-files.sh` 文件并向其中写入如下内容：

```bash
# 将需要的文件复制到 /etc/etcd/cert 目录下，然后进行分发
mkdir -p /etc/etcd/cert
cp etcd*pem ca.pem /etc/etcd/cert/

CONTROL_PLANE_IPS="172.16.99.26 172.16.99.27"
for host in ${CONTROL_PLANE_IPS}; do
    echo ">>> $host"
    ssh root@$host "mkdir -p /etc/etcd/cert"
    scp /etc/etcd/cert/* root@$host:/etc/etcd/cert
done
```

5. 创建 systemd 模板文件

host1 中执行如下命令：

```bash
cat > /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd
ExecStart=/root/etcd \\
  --data-dir="/var/lib/etcd/default.etcd" \\
  --name="host1" \\
  --cert-file="/etc/etcd/cert/etcd.pem" \\
  --key-file="/etc/etcd/cert/etcd-key.pem" \\
  --trusted-ca-file="/etc/etcd/cert/ca.pem" \\
  --peer-cert-file="/etc/etcd/cert/etcd.pem" \\
  --peer-key-file="/etc/etcd/cert/etcd-key.pem" \\
  --peer-trusted-ca-file="/etc/etcd/cert/ca.pem" \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls="https://172.16.99.25:2380" \\
  --initial-advertise-peer-urls="https://172.16.99.25:2380" \\
  --listen-client-urls="https://172.16.99.25:2379,http://127.0.0.1:2379" \\
  --advertise-client-urls="https://172.16.99.25:2379" \\
  --initial-cluster-token="etcd-cluster-0" \\
  --initial-cluster="host1=https://172.16.99.25:2380,host2=https://172.16.99.26:2380,host3=https://172.16.99.27:2380" \\
  --initial-cluster-state="new" \\
  --auto-compaction-mode="periodic" \\
  --auto-compaction-retention="1" \\
  --max-request-bytes="33554432" \\
  --quota-backend-bytes="6442450944" \\
  --heartbeat-interval="250" \\
  --election-timeout="2000"
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

其中的参数说明：

- `data-dir`：指定数据目录，在启动服务前需要创建这个目录。
- `name`：节点名称，注意每个节点名称都应该各不相同，后面两个配置文件也需要修改 name 的值。另外 name 的参数值必须位于 initial-cluster 列表中。
- `cert-file` 和 `key-file`：指定 etcd server 与 client 通信时使用的证书和私钥。
- `trusted-ca-file`：签名 client 证书的 CA 证书，用于验证 client 证书。
- `peer-cert-file` 和 `peer-key-file`：etcd 与 peer 通信使用的证书和私钥。
- `peer-trusted-ca-file`：签名 peer 证书的 CA 证书，用于验证 peer 证书。
- `listen-peer-urls`、`initial-advertise-peer-urls`、`listen-client-urls`、`advertise-client-urls`、`initial-cluster`：需要根据实际的 IP 地址进行修改。在后面两个配置文件中也需要修改这些值。

host2 中执行如下命令：

```bash
cat > /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd
ExecStart=/root/etcd \\
  --data-dir="/var/lib/etcd/default.etcd" \\
  --name="host2" \\
  --cert-file="/etc/etcd/cert/etcd.pem" \\
  --key-file="/etc/etcd/cert/etcd-key.pem" \\
  --trusted-ca-file="/etc/etcd/cert/ca.pem" \\
  --peer-cert-file="/etc/etcd/cert/etcd.pem" \\
  --peer-key-file="/etc/etcd/cert/etcd-key.pem" \\
  --peer-trusted-ca-file="/etc/etcd/cert/ca.pem" \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls="https://172.16.99.26:2380" \\
  --initial-advertise-peer-urls="https://172.16.99.26:2380" \\
  --listen-client-urls="https://172.16.99.26:2379,http://127.0.0.1:2379" \\
  --advertise-client-urls="https://172.16.99.26:2379" \\
  --initial-cluster-token="etcd-cluster-0" \\
  --initial-cluster="host1=https://172.16.99.25:2380,host2=https://172.16.99.26:2380,host3=https://172.16.99.27:2380" \\
  --initial-cluster-state="new" \\
  --auto-compaction-mode="periodic" \\
  --auto-compaction-retention="1" \\
  --max-request-bytes="33554432" \\
  --quota-backend-bytes="6442450944" \\
  --heartbeat-interval="250" \\
  --election-timeout="2000"
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

host3 中执行如下命令：

```bash
cat > /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd
ExecStart=/root/etcd \\
  --data-dir="/var/lib/etcd/default.etcd" \\
  --name="host3" \\
  --cert-file="/etc/etcd/cert/etcd.pem" \\
  --key-file="/etc/etcd/cert/etcd-key.pem" \\
  --trusted-ca-file="/etc/etcd/cert/ca.pem" \\
  --peer-cert-file="/etc/etcd/cert/etcd.pem" \\
  --peer-key-file="/etc/etcd/cert/etcd-key.pem" \\
  --peer-trusted-ca-file="/etc/etcd/cert/ca.pem" \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls="https://172.16.99.27:2380" \\
  --initial-advertise-peer-urls="https://172.16.99.27:2380" \\
  --listen-client-urls="https://172.16.99.27:2379,http://127.0.0.1:2379" \\
  --advertise-client-urls="https://172.16.99.27:2379" \\
  --initial-cluster-token="etcd-cluster-0" \\
  --initial-cluster="host1=https://172.16.99.25:2380,host2=https://172.16.99.26:2380,host3=https://172.16.99.27:2380" \\
  --initial-cluster-state="new" \\
  --auto-compaction-mode="periodic" \\
  --auto-compaction-retention="1" \\
  --max-request-bytes="33554432" \\
  --quota-backend-bytes="6442450944" \\
  --heartbeat-interval="250" \\
  --election-timeout="2000"
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

在阿里云上开放 3 个节点的 2379 和 2380 端口。

然后在 3 个节点分别启动 etcd，注意第一次启动 etcd 进程可能会卡顿，因为需要等待其它 etcd 节点启动进程加入集群：

```bash
mkdir -p /var/lib/etcd/default.etcd
systemctl daemon-reload && systemctl enable etcd && systemctl restart etcd
```

6. 查看节点健康状态

在 host1 节点执行：

```bash
for ip in 172.16.99.{25,26,27}; do
    ETCDCTL_API=3 /root/etcdctl \
    --endpoints=https://${ip}:2379  \
    --cacert=/etc/etcd/cert/ca.pem \
    --cert=/etc/etcd/cert/etcd.pem \
    --key=/etc/etcd/cert/etcd-key.pem \
    endpoint health; 
done
```

结果如下所示：

```text
https://172.16.99.25:2379 is healthy: successfully committed proposal: took = 9.507846ms
https://172.16.99.26:2379 is healthy: successfully committed proposal: took = 9.666582ms
https://172.16.99.27:2379 is healthy: successfully committed proposal: took = 9.178718ms
```

查看当前的 leader：

```bash
/root/etcdctl \
  -w table --cacert=/etc/etcd/cert/ca.pem \
  --cert=/etc/etcd/cert/etcd.pem \
  --key=/etc/etcd/cert/etcd-key.pem \
  --endpoints="172.16.99.25:2379,172.16.99.26:2379,172.16.99.27:2379" endpoint status
```

结果如下所示：

```text
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|     ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 172.16.99.25:2379 | 9cbf3b477c8a0cd6 |   3.4.9 |   20 kB |      true |      false |        52 |          9 |                  9 |        |
| 172.16.99.26:2379 | ef38d24486369605 |   3.4.9 |   20 kB |     false |      false |        52 |          9 |                  9 |        |
| 172.16.99.27:2379 | 8f4a94ebb3df1241 |   3.4.9 |   20 kB |     false |      false |        52 |          9 |                  9 |        |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

7. 创建一个 kube-master 节点，并使用 etcd 集群

在 host1 主机上执行如下脚本，将访问 etcd 所需的证书复制到 kube-master 节点上：

```bash
CONTROL_PLANE_IPS="172.16.99.28"
for host in ${CONTROL_PLANE_IPS}; do
    echo ">>> $host"
    ssh root@$host "mkdir -p /etc/etcd/cert"
    scp /etc/etcd/cert/* root@$host:/etc/etcd/cert
done
```

接下来在 kube-master 节点执行如下命令，对于 kube-master 节点的基本配置在上一个实验中有，这里不再赘述。这里依然以 1.17.0 版本作为示例。

下载镜像，新建 `image.sh` 文件，写入如下内容：

```bash
#!/bin/bash
url=registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes
version=v1.17.0
images=(`kubeadm config images list --kubernetes-version=$version|awk -F '/' '{print $2}'`)
for imagename in ${images[@]} ; do
  docker pull $url/$imagename
  docker tag $url/$imagename k8s.gcr.io/$imagename
  docker rmi -f $url/$imagename
done
```

然后下载镜像：

```bash
bash image.sh
docker images
```

新建 `kubeadm-config.yaml` 初始化配置文件：

```bash
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.17.0
# 使用外部 etcde 配置
etcd:
  external:
    endpoints:
    - "https://172.16.99.25:2379"
    - "https://172.16.99.26:2379"
    - "https://172.16.99.27:2379"
    caFile: "/etc/etcd/cert/ca.pem"
    certFile: "/etc/etcd/cert/etcd.pem"
    keyFile: "/etc/etcd/cert/etcd-key.pem"
networking:
  podSubnet: "10.244.0.0/16"
```

进行初始化：

```bash
$ kubeadm init --config=kubeadm-config.yaml
...
...
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.99.28:6443 --token xf1gn0.g1q819z2jljv2w10 \
    --discovery-token-ca-cert-hash sha256:9c83e5097f4da46de01e0fe4db90472cea9c726be4611f24a7df785e68004705
```

加载环境变量：

```bash
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source .bash_profile
```

安装 flannel 网络：

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
```

查看是否安装成功：

```bash
$ kubectl get nodes
NAME                      STATUS   ROLES    AGE    VERSION
izbp1bm7y86rsdg87xegn8z   Ready    master   3m9s   v1.17.0
$ kubectl get pods -n kube-system
NAME                                              READY   STATUS    RESTARTS   AGE
coredns-6955765f44-clf8v                          1/1     Running   0          3m10s
coredns-6955765f44-dpdsj                          1/1     Running   0          3m10s
kube-apiserver-izbp1bm7y86rsdg87xegn8z            1/1     Running   0          3m4s
kube-controller-manager-izbp1bm7y86rsdg87xegn8z   1/1     Running   0          3m4s
kube-flannel-ds-amd64-g72sf                       1/1     Running   0          54s
kube-proxy-l7757                                  1/1     Running   0          3m10s
kube-scheduler-izbp1bm7y86rsdg87xegn8z            1/1     Running   0          3m4s
```

验证：

```bash
$ kubectl taint nodes izbp1bm7y86rsdg87xegn8z node-role.kubernetes.io/master-
node/izbp1bm7y86rsdg87xegn8z untainted
$ kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-86c57db685-x2xfz   1/1     Running   0          39s
```

