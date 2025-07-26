# 3. Namespaces, Multi-Container Pod, Fetch Nodes Information in JSON

# Q6
Create a pod called pod-multi with two containers, as given below:

Container 1:

name: container1

image: nginx

Container 2:

name: container2

image: busybox

command: sleep 4800

```bash
kubectl run pod-multi --image=nginx --dry-run=client -o yaml > q6.yaml
vi q6.yaml
kubectl apply -f q6.yaml
k get po pod-multi -o yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-multi
spec:
  containers:
  - image: nginx
    name: container1
  - image: busybox
    name: container2
    command: ['sh', '-c', 'sleep 4800']
```

# Q7

Create a pod called test-pod in the "custom" namespace
Belonging to the test environment (env=test) and backend tier (tier=backend)
Image: nginx:1.17



```bash
kubectl create ns custom
kubectl run test-pod -n custom --image=nginx:1.17 --labels="env=test,tier=backend"
k get po -n custom
```

# Q8

Get the node node01 in JSON format and store it in a file at:

```pgsql
./node-info.json
```
Answer

```bash
kubectl get no node01 -o json > ./node-info.json
```