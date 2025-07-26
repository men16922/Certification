# 12. Kubernetes Context - 2

# Q20
There are various Pods in all namespaces.
Write a command into
/opt/course/5/find_pods.sh
which lists all Pods sorted by their AGE (metadata.creationTimestamp).

Write a second command into
/opt/course/5/find_pods_uid.sh
which lists all Pods sorted by field metadata.uid.
Use kubectl sorting for both commands.


```bash
mkdir -p /opt/course/5
echo "kubectl get pods -A --sort-by='.metadata.creationTimestamp'" > /opt/course/5/find_pods.sh
chmod +x /opt/course/5/find_pods.sh
bash /opt/course/5/find_pods.sh

echo "kubectl get pods -A --sort-by='.metadata.uid'" > /opt/course/5/find_pods_uid.sh
chmod +x /opt/course/5/find_pods_uid.sh
bash /opt/course/5/find_pods_uid.sh
```

# Q21
Weightage: 11%

Ssh into the controlplane node with
ssh cluster1-controlplane1.
Check how the controlplane components
kubelet, kube-apiserver, kubescheduler, kube-controller-manager and etcd
are started/installed on the controlplane node.
Also find out the name of the DNS application and how it's started/installed on the controlplane node.

Write your findings into file:
/opt/course/8/controlplane-components.txt

The file should be structured like:

```graphql
# /opt/course/8/controlplane-components.txt
kubelet: [TYPE]  
kube-apiserver: [TYPE]  
kube-scheduler: [TYPE]  
kube-controller-manager: [TYPE]  
etcd: [TYPE]  
dns: [TYPE] [NAME]
Choices of [TYPE] are:
not-installed, process, static-pod, pod
```



- [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/#the-kubelet-drop-in-file-for-systemd](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/#the-kubelet-drop-in-file-for-systemd)

```bash
mkdir -p /opt/course/8

find /etc/systemd/system/ | grep kube
find /etc/kubernetes/manifests/

kubectl get po -n kube-system
kubectl get deploy -n kube-system

vi /opt/course/8/controlplane-components.txt

```

```
kubelet: process
kube-apiserver: static-pod
kube-scheduler: static-pod
kube-controller-manager: static-pod
etcd: static-pod
dns: pod coredns
```