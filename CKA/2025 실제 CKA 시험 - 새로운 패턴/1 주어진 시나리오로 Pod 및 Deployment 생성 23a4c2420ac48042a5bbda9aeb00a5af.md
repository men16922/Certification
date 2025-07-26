# 1. 주어진 시나리오로 Pod 및 Deployment 생성

# Q1
Create a new pod called web-pod with image busybox
Allow the pod to be able to set system_time

The container should sleep for 3200 seconds

- [https://kubernetes.io/docs/tasks/configure-pod-container/security-context/](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

```bash
k run web-pod --image=busybox --command sleep 3200 --dry-run=client -o yaml > web-pod.yaml

```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: web-pod
  name: web-pod
spec:
  containers:
  - command:
    - sleep
    - "3200"
    image: busybox
    name: web-pod
    securityContext:
      capabilities:
        add: ["SYS_TIME"]
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

# Q2
Create a new deployment called myproject, with image nginx:1.16 and 1 replica.
Next upgrade the deployment to version 1.17 using rolling update

Make sure that the version upgrade is recorded in the resource annotation.

```bash
kubectl create deploy myproject --image=nginx:1.16 --replicas=1
kubectl set image deploy/myproject nginx=nginx:1.17

kubectl rollout history deploy/myproject
```