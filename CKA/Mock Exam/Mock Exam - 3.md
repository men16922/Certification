# Q1

You are an administrator preparing your environment to deploy a Kubernetes cluster using kubeadm. Adjust the following network parameters on the system to the following values, and make sure your changes persist reboots:

net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1


net.ipv4.ip_forward is set to 1

net.bridge.bridge-nf-call-iptables is set to 1

```shell
cat /proc/sys/net/ipv4/ip_forward
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

# Q2
Create a new service account with the name pvviewer. Grant this Service account access to list all PersistentVolumes in the cluster by creating an appropriate cluster role called pvviewer-role and ClusterRoleBinding called pvviewer-role-binding.
Next, create a pod called pvviewer with the image: redis and serviceAccount: pvviewer in the default namespace.


ServiceAccount: pvviewer

ClusterRole: pvviewer-role

ClusterRoleBinding: pvviewer-role-binding

Pod: pvviewer

Is the pod configured to use ServiceAccount pvviewer?

```shell
kubectl create sa pvviewer
kubectl create clusterrole pvviewer-role --verb=list --resource=persistentvolumes
kubectl create rolebinding pvviewer-role-binding --clusterrole=pvviewer-role --serviceaccount=default:pvviewer
kubectl run pvviewer --image=redis --dry-run=client -o yaml > q2.yaml
vi q2.yaml
kubectl apply -f q2.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvviewer
spec:
  serviceAccountName: pvviewer
  containers:
  - image: redis
    name: pvviewer
```

# Q3
Create a StorageClass named rancher-sc with the following specifications:

The provisioner should be rancher.io/local-path.
The volume binding mode should be WaitForFirstConsumer.
Volume expansion should be enabled.


StorageClass rancher-sc is present

Provisioner is rancher.io/local-path

VolumeBindingMode is WaitForFirstConsumer

```shell
vi q3.yaml
kubectl apply -f q3.yaml
```
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rancher-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: rancher.io/local-path
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

# Q4
Create a ConfigMap named app-config in the namespace cm-namespace with the following key-value pairs:

ENV=production
LOG_LEVEL=info

Then, modify the existing Deployment named cm-webapp in the same namespace to use the app-config ConfigMap by setting the environment variables ENV and LOG_LEVEL in the container from the ConfigMap.


ConfigMap app-config is created

Deployment uses the app-config ConfigMap for variable ENV and LOG LEVEL

Are the environment variables reflected in the deployment?

ConfigMap has proper ENV value

ConfigMap has proper LOG_LEVEL value

```shell
kubectl create configmap app-config -n cm-namespace --from-literal=ENV=production --from-literal=LOG_LEVEL=info
kubectl get configmap app-config -n cm-namespace -o yaml
kubectl edit deploy cm-webapp -n cm-namespace
kubectl get deploy cm-webapp -n cm-namespace -o yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
  creationTimestamp: "2025-07-23T17:52:45Z"
  generation: 2
  labels:
    app: cm-webapp
  name: cm-webapp
  namespace: cm-namespace
  resourceVersion: "4421"
  uid: 7cf9e724-e501-45a0-a859-6a0599b9c356
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: cm-webapp
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: cm-webapp
    spec:
      containers:
      - envFrom:
        - configMapRef:
            name: app-config
        image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2025-07-23T17:52:49Z"
    lastUpdateTime: "2025-07-23T17:52:49Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2025-07-23T17:52:45Z"
    lastUpdateTime: "2025-07-23T17:56:58Z"
    message: ReplicaSet "cm-webapp-787f95558b" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 2
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1
```

# Q5
Create a PriorityClass named low-priority with a value of 50000. A pod named lp-pod exists in the namespace low-priority. Modify the pod to use the priority class you created. Recreate the pod if necessary.


Is the PriorityClass low-priority created?

Low priority class value is set properly to 50000

Pod lp-pod uses the low-priority PriorityClass

```shell
vi q5.yaml
kubectl apply -f q5.yaml

kubectl get pod lp-pod -n low-priority -o yaml > lp-pod.yaml
vi lp-pod.yaml

kubectl delete po lp-pod -n low-priority
kubectl apply -f lp-pod.yaml
kubectl describe po lp-pod -n low-priority
```

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name:  low-priority
value: 50000
preemptionPolicy: Never
globalDefault: false
description: "This priority class will not cause other pods to be preempted."
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
  name: lp-pod
  namespace: low-priority
spec:
  priorityClassName: low-priority
  containers:
  - image: nginx
    name: lp-pod
```

# Q6
We have deployed a new pod called np-test-1 and a service called np-test-service. Incoming connections to this service are not working. Troubleshoot and fix it.
Create NetworkPolicy, by the name ingress-to-nptest that allows incoming connections to the service over port 80.


Important: Don't delete any current objects deployed.

Important: Don't Alter Existing Objects!

NetworkPolicy: Is it applied to all sources (Incoming traffic from all pods)?

NetworkPolicy: Is the port correct?

NetworkPolicy: Is it applied to the correct Pod?

```shell
vi q6.yaml
kubectl apply -f q6.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-to-nptest
spec:
  podSelector:
    matchLabels:
      role: np-test-1
  policyTypes:
  - Ingress
  ingress:
  - from:
    ports:
    - protocol: TCP
      port: 80
```

# Q7
Taint the worker node node01 to be Unschedulable. Once done, create a pod called dev-redis, image redis:alpine, to ensure workloads are not scheduled to this worker node. Finally, create a new pod called prod-redis and image: redis:alpine with toleration to be scheduled on node01.


key: env_type, value: production, operator: Equal and effect: NoSchedule

key = env_type

value = production

effect = NoSchedule

Is pod 'dev-redis' (no tolerations) not scheduled on node01?

Is the 'prod-redis' to running on node01?

```shell
kubectl taint nodes node01 env_type=production:NoSchedule
kubectl run dev-redis --image=redis:alpine
kubectl run prod-redis --image=redis:alpine --dry-run=client -o yaml > prod-redis.yaml
vi prod-redis.yaml
kubectl apply -f prod-redis.yaml
kubectl get po -o wide
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-redis
spec:
  containers:
  - image: redis:alpine
    name: prod-redis
  tolerations:
  - key: "env_type"
    value: "production"
    operator: "Equal"
    effect: "NoSchedule"
```

# Q8
A PersistentVolumeClaim named app-pvc exists in the namespace storage-ns, but it is not getting bound to the available PersistentVolume named app-pv.

Inspect both the PVC and PV and identify why the PVC is not being bound and fix the issue so that the PVC successfully binds to the PV. Do not modify the PV resource.


Is PVC correctly bound to PV?

```shell
kubectl get pv,pvc -n storage-ns
kubectl get pv app-pv -o yaml -n storage-ns
kubectl get pvc app-pvc -o yaml -n storage-ns > app-pvc.yaml
kubectl delete pvc app-pvc -n storage-ns
kubectl apply -f app-pvc.yaml

```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
  namespace: storage-ns
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```