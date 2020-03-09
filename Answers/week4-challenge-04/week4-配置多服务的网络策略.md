# 挑战：配置多服务的网络策略

networkpolicy.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: staging.db
  namespace: staging
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: backend
        - namespaceSelector:
            matchLabels:
              role: admin
      ports:
        - protocol: TCP
          port: 5432
  policyTypes:
    - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: staging.backend
  namespace: staging
spec:
  podSelector:
    matchLabels:
      role: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
        - namespaceSelector:
            matchLabels:
              role: admin
      ports:
        - protocol: TCP
          port: 80
  policyTypes:
    - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: staging.frontend
  namespace: staging
spec:
  podSelector:
    matchLabels:
      role: frontend
  ingress:
    - from:
        - podSelector: {}
        - namespaceSelector:
            matchLabels:
              role: admin
      ports:
        - protocol: TCP
          port: 80
  policyTypes:
    - Ingress
```
