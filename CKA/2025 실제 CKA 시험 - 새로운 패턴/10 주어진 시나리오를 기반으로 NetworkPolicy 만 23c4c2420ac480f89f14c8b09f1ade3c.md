# 10. 주어진 시나리오를 기반으로 NetworkPolicy 만들기

# Q18
Weightage: 11%

Create a Network Policy named "appychip" in default namespace
There should be two types, ingress and egress.

The ingress should block traffic from an IP range of your choice except some other IP range.
Should also have namespace and pod selector.
Ports for ingress policy should be 6379

For Egress, it should allow traffic to an IP range of your choice on 5978 port.

- [https://kubernetes.io/docs/concepts/services-networking/network-policies/](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

```bash
vi q18.yaml
kubectl apply -f q18.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: appychip
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978

```