# 挑战：配置 Service Account 访问 Kubernetes API

创建名为 `shiyanlou-api-service-account` 的服务账户：

```bash
$ kubectl create serviceaccount shiyanlou-api-service-account
serviceaccount/shiyanlou-api-service-account created
$ kubectl get sa
NAME                            SECRETS   AGE
shiyanlou-api-service-account   1         8m20s
default                         1         200d
```

为创建的服务账户设置 ClusterRole 和 ClusterRoleBinding，这里默认配置了所有的资源访问权限。

```bash
# 配置 ClusterRole 和 ClusterRoleBinding
$ touch clusterRole.yaml

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: api-access
rules:
  -
    apiGroups:
      - ""
      - apps
      - autoscaling
      - batch
      - extensions
      - policy
      - rbac.authorization.k8s.io
    resources:
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["*"]
  - nonResourceURLs: ["*"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: api-access
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: api-access
subjects:
- kind: ServiceAccount
  name: shiyanlou-api-service-account
  namespace: default

# 执行创建
$ kubectl create -f clusterRole.yaml
clusterrole.rbac.authorization.k8s.io/api-access created
clusterrolebinding.rbac.authorization.k8s.io/api-access created
```

现在这个服务账户已经拥有了对应的集群访问权限，接下来需要获取 Service Account 对应 Secrets 的 token：

```bash
$ sudo apt install -y jq
# 获取 secrets 的名称
$ kubectl get serviceaccount shiyanlou-api-service-account  -o json | jq -Mr '.secrets[].name'
api-service-account-token-qmwlr
# 获取对应的 token，需要注意获取到的 token 还需要经过 base64 解密以后才能使用
$ kubectl get secrets shiyanlou-api-service-account-token-qmwlr -o json | jq -Mr '.data.token' | base64 -d
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImFwaS1zZXJ2aWNlLWFjY291bnQtdG9rZW4tMnNzcnAiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiYXBpLXNlcnZpY2UtYWNjb3VudCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjQ3OTViNWFjLWNiYjUtNDc0Ni05OTI3LWEyYmJmMjM4ZjBlNSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmFwaS1zZXJ2aWNlLWFjY291bnQifQ.cl-YLDeOs0zF8KohaUc8jLPM6REms8GMm5eMhC9tNFJxNCNsxX9NEFd1brTsJO_drPo513r7gJ0SHRhIPaYZYSJAP1svO8MtnLCbMgT9fuIgjMWzlkr89T6vnTIuQazveUbJaNkx7mxbCHvoME7ci14Un3hDwNDcJU_m3oypRL4KWdNbNYVuepeHQQTUSxO_-GAtt88UBF1x4yH5cxjr4fkqlnHr-pp1k-6eBbVKK0GTo8-QAXTm5566DCjdgslHze6CXn0XctcYkyCYBMi91FCh9wx-EAp5TvfesGgpbemHkw8wXrSOuUYKujbwGzV5J4jXDf2VPvy7ApnSzw_YJQ
```

进行测试：

```bash
# 获取集群中 kubernetes 对应的 endpoint，也就是 Server API 的地址
$ kubectl get endpoints | grep kubernetes
kubernetes   10.192.0.2:6443   200d
# 测试，访问非资源型URL：healthz/ping，<TokenOfSecret> 就是通过上面命令获取到的 token
$ curl -k 'https://10.192.0.2:6443/healthz/ping' -H 'Authorization: Bearer <TokenOfSecret>'
ok
# 测试，访问命名空间资源：api/v1/namespaces，<TokenOfSecret> 就是通过上面命令获取到的 token
$ curl -k 'https://10.192.0.2:6443/api/v1/namespaces' -H 'Authorization: Bearer <TokenOfSecret>'
{
  "kind": "NamespaceList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces",
    "resourceVersion": "1438"
  },
  "items": [
    {
      "metadata": {
        "name": "default",
        "selfLink": "/api/v1/namespaces/default",
        "uid": "462d8172-6d21-4562-b13f-4b2cbf8db62f",
        "resourceVersion": "148",
        "creationTimestamp": "2019-08-18T01:35:53Z"
      },
...
```
