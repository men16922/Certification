# 10. WordPress Application Some Pods are Nou Up and running 2025
Weightage: 9%

You manage a WordPress application. Some pods are not up and running.
Adjust all Pod resource requests as follows:

Divide node resources evenly across all 3 pods.

Give each Pod a fair share of CPU and memory.

Add enough overhead to keep the node stable.

Note - Use exact same requests for both containers and init containers.
Scale down the WordPress deployment to 0 replicas while updating the resource requests.

After updates, confirm:

WordPress keeps 3 replicas.

All Pods are running and ready.

```bash
vi wordpress-deploy.yaml
kubectl apply -f wordpress-deploy.yaml
kubectl scale deploy wordpress --replicas=0

kubectl describe node controlplane
kubectl patch deployment wordpress -p '{"spec": {"template": {"spec": {"initContainers": [{"name": "init-wordpress", "resources": {"requests": {"cpu": "100m", "memory": "100Mi"}}}]}}}}'
kubectl patch deployment wordpress -p '{"spec": {"template": {"spec": {"containers": [{"name": "wordpress", "resources": {"requests": {"cpu": "100m", "memory": "100Mi"}}}]}}}}'
kubectl scale deploy wordpress --replicas=3
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 3
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      initContainers:
      - name: init-wordpress
        image: busybox
        command: ["sh", "-c", "echo Initializing && sleep 5"]
      containers:
      - name: wordpress
        image: wordpress:latest
        ports:
        - containerPort: 80

```

![image.png](image%2040.png)