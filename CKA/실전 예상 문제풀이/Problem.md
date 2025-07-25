# 실전 문제
[TOC]

### ❓1. Retrieve Error Messages from a Container Log

- Cluster: kubectl config use-context **hk8s**

---

In the `customera` namespace, check the log for the nginx container in the `custom-app` Pod.

Save the lines which contain the text “error” to the file /data/CKA/errors.txt.

```bash
kubectl get po -n customera
kubectl logs custom-app -n customera | grep error > /data/CKA/errors.txt
cat /data/CKA/errors.txt
```

### ❓2. Node Troubleshooting

- Cluster: kubectl config use-context **hk8s**

---

A Kubernetes worker node, named hk8s-worker2 is in state NotReady.
Investigate why this is the case, and perform any appropriate steps to bring the node to a Ready state, ensuring that any changes are made permanent.

---

```bash
kubectl get no
kubectl get po -n kube-system

ssh hk8s-worker2
systemctl status kubelet
systemctl status docker
sudo systemctl enable --now kubelet

exit
kubectl get no
```

### ❓3. Count the Number of Nodes That Are Ready to Run Normal Workloads

- Cluster: kubectl config use-context **hk8s**

---

Determine how many nodes in the cluster are ready to run **normal workloads** (i.e., workloads that **do not have any special tolerations**).

Output this number to the file  /var/CKA2022/count.txt

---

```bash
kubectl get no -o wide | grep Ready
kubectl describe no hk8s-master | grep -i taint
kubectl describe no hk8s-worker1 | grep -i taint
kubectl describe no hk8s-worker2 | grep -i taint

echo "2" > /var/CKA2022/count.txt
cat /var/CKA2022/count.txt
```

### **❓4. Management Node**

- Cluster: kubectl config use-context **k8s**

---

**Set the node named k8s-worker1 as unavailable and reschedule all the pods running on it.**

---

```bash
kubectl drain k8s-worker1 --ignore-daemonsets --force --delete-emptydir-data
kubectl get nodes
kubectl get pods
```

### **❓5. ETCD backup & restore**

<aside>
📌 ETCD 백업 및 리스토어 문제 유형이 바뀌어서 바뀐 문제로 출제했습니다. 학습 영상을 통해 학습하시면 무난히 풀 수 있습니다.
변경 사항 : 학습 영상에서는 k8s-master로 접속해서 백업을 하고, 리스토어를 진행하였으나, 최근 유형에서는 
- 백업 : console  etcdctl 명령을 실행해서 master 의 etcd 데이터를 스냅샷 해서 콘솔의 특정 디렉토리로 저장합니다.
- 리소트어 : 리스토어는 주어진 snapshot.db 파일을 masterf로 전달(또는 마스터에 저장되어 있음)하고,  master 로 ssh 원격 로그인 후 etcdctl  명령으로 restore합니다. 리스토어 후  etcd.yaml 도 당연히 수정하고 파드 리스타스 되었는지도 확인합니다.

문제 해결 과정을 정리하면 :
**1. console에 etcdctl이 설치되어있습니다. console에서 k8s-master의 etcd를 snapshot save합니다.
인증서 파일 모두 console에 존재합니다. 예를 들어 /home/ubuntu/key 디렉토리에 인증서 모두 저장됨
백업 명령에서 endpoint만 바꿔주면 됩니다.**

    **sudo ETCDCTL_API=3 etcdctl  --endpoints=https://10.0.2.10:2379 \
        --cacert=/data/cka/ca.crt   --cert=/data/cka/server.crt  --key=/data/cka/server.key \
         snapshot save /data/previous.db
2. restore 문제는 백업을 실행한 클러스터가 아닌 다른 클러스터에 snapshot 파일을 저장해서 복원해야 합니다.
   scp /data/cka/etcd-snapshot-previous.db  k8s-master:~
   ssh k8s-master
   sudo ETCDCTL_API=3  etcdctl --data-dir=/var/lib/etcd-restore snapshot restore** etcd-snapshot-previous.db
   sudo vi /etc/kubernetes/manifests/etcd.yaml

**3. docker  대신 crictl   명령으로 etcd  동작되는지 확인하세요.**
sudo crictl ps    # 실행하면 etcd id 표시됩니다. 그거 가지고 logs 확인하면 됩니다. 아니면 더 쉽게 console에서 2-3분 후에 kubectl 명령을 실행해보시면 됩니다.
sudo crictl  logs etcd'sID
exit

</aside>

### **❓console에서  k8s master의 etcd backup & restore**

- 사전 환경 구성

    ```bash
    # console에서 k8s-master에 저장된 인증서와 백업 파일을 /data/cka 폴더에 저장한 후 기출문제를 풀어보세요.
    # consle에서 실행합니다.
    sudo -i
    mkdir -p /data/cka
    cd /data/cka/
    scp k8s-master:/etc/kubernetes/pki/etcd/ca.crt .
    scp k8s-master:/etc/kubernetes/pki/etcd/server.crt .
    scp root@k8s-master:/etc/kubernetes/pki/etcd/server.key .
    scp root@k8s-master:/data/etcd-snapshot-previous.db .
    ```

- 작업 클러스터 :  **k8s**
  First, create a snapshot of the existing etcd instance running at `https://<k8s-master's IP>:2379`, saving the snapshot to `/data/etcd-snapshot.db`.
  Next, restore an existing, previous snapshot located at `/data/etcd-snapshot-previous.db`.

The following TLS certificates/key are supplied for connecting to the server with etcdctl:
CA certificate: `/data/cka/ca.crt`
Client certificate: `/data/cka/server.crt`
Client key: `/data/cka/server.key`

```bash
ETCDCTL_API=3 etcdctl -h 

ETCDCTL_API=3 etcdctl --endpoints=https://k8s-master:2379 \
  --cacert='/data/cka/ca.crt' --cert='/data/cka/server.crt' --key='/data/cka/server.key' \
  snapshot save '/data/etcd-snapshot.db'

ssh k8s-master
ETCDCTL_API=3 etcdctl --data-dir '/var/lib/etcd-restore' snapshot restore /data/CKA/etcd-snapshot-previous.db
sudo vi /etc/kubernetes/manifests/etcd.yaml

crictl ps | grep etcd
sudo crictl logs 'etcd-id'
exit

kubectl get nodes
```

### **❓6. Cluster Upgrade - only Master**

<aside>
📌 클러스터 업그레이드는 버전이 계속 바뀝니다. 시험 버전이 바뀌니 어쩔수 없겠죠? 먼저 학습 영상에서 업그레이드를 어떤 단계로 진행하는지, 또 dos 를 어떻게 활용하는지 보시면 사실 버전에 영향받지 않고 업그레이드 가능합니다.  docs의 순서대로 진행하세요. 버전의 숫자만 다르게 넣으면 업그레이드 됩니다. 
기출문제 풀이시 참조 : 2024.03월 업데이트한 실습 환경 구성 완료된 클러스터에는 1.28.0 버전이 설치되어 있고, 업그레이드는 1.28.2 로 하시면 됩니다.

</aside>

- 작업 클러스터 : kubectl config use-context **k8s**

---

upgrade system : hk8s-master
Given an existing Kubernetes cluster running version `1.28.0`,
upgrade all of the Kubernetes control plane and node components on the master node only to version `1.28.2`.
Be sure to `drain` the `master` node before upgrading it and `uncordon` it after the upgrade.

---

```bash
# master
ssh k8s-master
sudo apt update
sudo apt-cache madison kubeadm

sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.32.7-1.1' && \
sudo apt-mark hold kubeadm

kubeadm version
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.32.7

kubectl drain k8s-master --ignore-daemonsets 

sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.32.7-1.1' kubectl='1.32.7-1.1' && \
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

kubectl uncordon k8s-master
kubectl get no

# worker
ssh k8s-worker1
sudo apt update
sudo apt-cache madison kubeadm

sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.32.7-1.1' && \
sudo apt-mark hold kubeadm

sudo kubeadm upgrade node

kubectl drain k8s-worker1 --ignore-daemonsets 

sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.32.7-1.1' kubectl='1.32.7-1.1' && \
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

kubectl uncordon k8s-worker1

## 안될경우
kubectl drain k8s-worker2 \
  --ignore-daemonsets \
  --force \
  --delete-emptydir-data

```

### **❓7.** Authentication and Authorization

- Cluster : **k8s**

---

Context You have been asked to create a new ClusterRole for a deployment pipeline and bind it to a specific ServiceAccount scoped to a specific namespace. 
**Task:**
  - Create a new ClusterRole named `deployment-clusterrole`, which only allows to **create** the following resource types: **Deployment StatefulSet DaemonSet**
  - Create a new ServiceAccount named `cicd-token` in the existing namespace `app-team1`. 
  - Bind the new ClusterRole `deployment-clusterrole` to the new ServiceAccount `cicd-token`, limited to the namespace `app-team1`.


```bash
kubectl create clusterrole deployment-clusterrole --verb=create --resource=deployment,statefulSet,daemonSet
kubectl describe clusterrole deployment-clusterrole

kubectl create serviceaccount cicd-token -n app-team1
kubectl create clusterrolebinding deployment-clusterrole-binding --clusterrole=deployment-clusterrole --serviceaccount=app-team1:cicd-token -n app-team1
kubectl describe clusterrolebinding deployment-clusterrole-binding -n app-team1
```

### **❓8.** Pod 생성하기

- 작업 클러스터 : kubectl config use-context **k8s**

---

 Create a new namespace and create a pod in the namespace

TASK:

- namespace name: cka-exam
- pod Name: pod-01
- image: busybox
- environment Variable:  CERT = "CKA-cert"
- command: /bin/sh
- args: -c "while true; do echo $(CERT); sleep 10;done"

```bash
kubectl create ns cka-exam
kubectl run pod-01 -n cka-exam --image=busybox --dry-run=client -o yaml > q8.yaml
vi q8.yaml
kubectl apply -f q8.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-01
  namespace: cka-exam
spec:
  containers:
  - image: busybox
    name: pod-01
    env:
    - name: CERT
      value: "CKA-cert"
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(CERT); sleep 10;done"]
```

### **❓9.** multi-container Pod 생성

- cluster : kubectl config use-context **hk8s**
- Create a pod with 4 containers running : nginx, redis and memcached
    - pod name: eshop-frontend
    - image: nginx
    - image: redis
    - image: memcached

```shell
kubectl config use-context hk8s
kubectl run eshop-frontend --image=nginx --dry-run=client -o yaml > q9.yaml
vi q9.yaml
kubectl apply -f q9.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: eshop-frontend
spec:
  containers:
  - image: nginx
    name: nginx-container
  - image: redis
    name: redis-container
  - image: memcached
    name: memcached-container

```

### **❓10.** Side-car Container Pod 실행

- 작업 클러스터 : kubectl config use-context **k8s**
- An existing Pod needs to be integrated into the `Kubernetes built-in logging architecture` (e.g. `kubectl logs`).
- Adding a streaming sidecar container is a good and common way to accomplish this requirement.
- Task:
    - Add a sidecar container named `sidecar`, using the busybox Image, to the existing Pod `eshop-cart-app`.
    - The new `sidecar` container has to run the following `command: /bin/sh -c "tail -n+1 -f /var/log/cart-app.log"`
    - Use a Volume, mounted at `/var/log`, to make the log file `cart-app.log` available to the `sidecar` container.
    - Don't modify the `cart-app`.

```bash
kubectl config use-context k8s
kubectl run eshop-cart-app --image=busybox --dry-run=client -o yaml > q10.yaml
vi q10.yaml
kubectl apply -f q10.yaml

```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: eshop-cart-app
spec:
  containers:
  - image: busybox
    name: cart-app
    command: ['/bin/sh', '-c', 'i=1;while :;do  echo -e "$i: Price: $((RANDOM % 10000 + 1))" >> /var/log/cart-app.log; i=$((i+1)); sleep 2; done']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - image: busybox
    name: sidecar
    command: ['/bin/sh', '-c', "tail -n+1 -f /var/log/cart-app.log"]
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - emptyDir: {}
    name: varlog
```

### ❓11. Pod Scale-out

- Cluster: kubectl config use-contex **k8s**

---

Expand the number of running Pods in "**eshop-order**" to 5.

- namespace: devops
- deployment: eshop-order
- replicas: **5**

```bash
kubectl scale deployment eshop-order --replicas=5 -n devops
kubectl get rs -n devops
```

### **❓12. Rolling Update**

- Cluster: kubectl config use-context **k8s**

---

**Create** a deployment as follows:

- TASK:
    - name: nginx-app
    - Using container nginx with version 1.11.10-alpine
    - The deployment should contain 3 replicas
- Next, deploy the application with new version **1.11.13-alpine**, by performing a **rolling update**
- Finally, rollback that update to the **previous version** 1.11.10-alpine
```bash
kubectl create deployment nginx-app --image=nginx:1.11.10-alpine --replicas=3
kubectl set image deployment/nginx-app nginx=nginx:1.11.13-alpine
kubectl rollout history deployment nginx-app
kubectl rollout history deployment/nginx-app --revision=2

kubectl rollout undo deployment/nginx-app
kubectl rollout history deployment nginx-app
```

### **❓13.** Network Policy with Namespace

- 작업 클러스터 : kubectl config use-context **k8s**

---

**Creat**e a new **NetworkPolicy** named allow-port-from-namespace in the existing **namespace devops**.

Ensure that the new NetworkPolicy allows Pods in namespace migops(using label **team=migops**) to connect to port 80 of Pods in namespace devops.

Further ensure that the new NetworkPolicy: does not allow access to Pods, which don't listen on port 80 does not allow access from Pods, which are not in namespace migops

```bash
kubectl get deployment -n devops
vi q13.yaml
kubectl apply -f q13.yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-from-namespace
  namespace: devops
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: migops
    ports:
    - protocol: TCP
      port: 80
```

### **❓14. Create a persistent volume**

- Cluster: kubectl config use-context **k8s**

---

**Create a persistent volume** with name app-config, of capacity 1Gi and access mode ReadWriteMany. The type of volume is hostPath and its location is /var/app-config.
```bash
vi q14.yaml
kubectl apply -f q14.yaml
```
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-config
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/var/app-config"
```

### **❓15. Deploy and Service**

- 작업 클러스터 : kubectl config use-context **k8s**

---

Reconfigure the **existing** deployment `front-end` and add a **port specification named** `http` exposing port `80/tcp` of the existing container nginx. Create a new service named `front-end-svc` exposing the container port http. Configure the new service to also expose the individual Pods via a `NodePort` on the nodes on which they are scheduled

```bash
kubectl get deploy -o yaml > front-end.yaml
kubectl delete deploy front-end
kubectl apply -f front-end.yaml

vi front-end-svc.yaml


```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: front-end
spec:
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx
        name: http
        ports:
        - containerPort: 80

```
```yaml
:q!apiVersion: v1
kind: Service
metadata:
  name: front-end-svc
spec:
  type: NodePort
  selector:
      run: nginx
  ports:
    - port: 80
      targetPort: 80
```

### **16.** DNS Lookup

- 작업 클러스터 : kubectl config use-context **k8s**

---

**Create a nginx pod** called nginx-resolver using image nginx, expose it internally with a service called nginx-resolver-service. Test that you are able to look up the service and pod names from within the cluster. Use the image: busybox:1.28 for dns lookup.

- Record results in /var/CKA2022/nginx.svc and /var/CKA2022/nginx.pod
- Pod: nginx-resolver created
- Service DNS Resolution recorded correctly
- Pod DNS resolution recorded correctly

```bash
kubectl run nginx-resolver --image=nginx
kubectl expose pod nginx-resolver --name=nginx-resolver-service --port 80

kubectl get po,svc -o wide

kubectl run test --image=busybox -it --rm --restart=Never nslookup 10.96.74.214  > /var/CKA2022/nginx.svc

kubectl run test --image=busybox -it --rm --restart=Never nslookup 10-32-0-9.default.pod.cluster.local  > /var/CKA2022/nginx.pod

cat /var/CKA2022/nginx.svc
cat /var/CKA2022/nginx.pod
```

### **❓17.** Application with PVC

- Cluster: kubectl config use-context **k8s**

---

Create a new PersistentVolumeClaim:

Name: pv-volume

Class: csi-hostpath-sc

Capacity: 10Mi

Create a new Pod which mounts the PersistentVolumeClaim as a volume:

Name: web-server

Image: nginx

Mount path: /usr/share/nginx/html

Configure the new Pod to have ReadWriteOnce access on the volume.
```bash
vi q17.yaml
kubectl apply -f q17.yaml
vi web-server.yaml
kubectl apply -f web-server.yaml
```
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-volume
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
  storageClassName: csi-hostpath-sc
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/usr/share/nginx/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: pv-volume
```