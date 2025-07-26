# 13. 2025 Network Policy YAML
There are two deployments. frontend and backend deployment.
frontend will be in frontend NS and backend will be in backend NS.
Apply the least permissive policy to have interaction between frontend and backend deployment.
Below are the 3 YAML file to apply network policy.
Choose either of them with least permission.

1 YAML - pod selector of all. type is ingress and pod selector with all
2 YAML - pod selector as well as namespace selector are present
3 YAML - pod selector, NS selector, POD CIDR


```bash
 # 2번째 정책이 가장 적절. 1번은 부족하고, 2번은 복잡함
 kubectl apply -f network-policy-2.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: open-backend-access
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from: []
      ports:
        - protocol: TCP
          port: 8080
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
          podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080

```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend-with-cidr
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
          podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - ipBlock:
            cidr: 192.168.1.0/24
      ports:
        - protocol: TCP
          port: 8080

```