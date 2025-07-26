# 2. Static Pod, Deployment, Replicas

# Q3

Create a new deployment called my-deployment.
Scale the deployment to 3 replicas.
Make sure desired number of pod always running.

```bash
kubectl create deploy my-deployment --image=nginx
kubectl scale deploy/my-deployment --replicas=3
kubectl get rs
```

# Q4

Deploy a pod called web-nginx using the image nginx:1.17
with the labels set to: tier=web-app.

```bash
kubectl run web-nginx --image=nginx:1.17 --labels tier=web-app
```

# Q5

Create a static pod on node01 called static-pod with image nginx
and you have to make sure that it is recreated/restarted automatically in case of any failure.

```bash
ssh node01
ps aux | grep kubelet
vi /etc/kubernetes/manifests/static-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-pod
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```