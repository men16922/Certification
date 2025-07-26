[TOC]

# Q1
Create a Pod named `debug-pod` in the `troubleshoot` namespace with the following specifications:
- Use the `busybox:1.35` image
- Set the command to sleep for 3600 seconds
- Add a label `env=debug`
- Set resource requests: CPU 100m, Memory 128Mi
- Set resource limits: CPU 200m, Memory 256Mi

```shell
kubectl create ns troubleshoot
kubectl run debug-pod -n troubleshoot --image=busybox:1.35 --dry-run=client -o yaml > q1.yaml
vi q1.yaml
kubectl apply -f q1.yaml
kubectl get po -n troubleshoot --show-labels
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  namespace: troubleshoot
  labels:
    env: debug
spec:
  containers:
  - name: debug-pod
    image: busybox:1.35
    command: ['sleep', '3600']
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
```

# Q2
Create a ConfigMap named `app-config` in the `production` namespace with the following key-value pairs:
- `database_url=postgresql://prod-db:5432/myapp`
- `log_level=INFO`
- `max_connections=100`

Then create a Pod named `config-consumer` that uses this ConfigMap as environment variables.

```shell
kubectl create ns production
kubectl create configmap app-config -n production \
  --from-literal=database_url=postgresql://prod-db:5432/myapp \
  --from-literal=log_level=INFO \
  --from-literal=max_connections=100

kubectl run config-consumer -n production --image=nginx --dry-run=client -o yaml > q2.yaml
vi q2.yaml
kubectl apply -f q2.yaml
kubectl exec -n production config-consumer -- env | grep -E "(database_url|log_level|max_connections)"
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-consumer
  namespace: production
spec:
  containers:
  - name: config-consumer
    image: nginx
    envFrom:
    - configMapRef:
        name: app-config
```

# Q3
Create a Secret named `db-credentials` in the `secure` namespace with the following data:
- `username=admin`
- `password=secretpassword123`

Mount this secret as a volume in a Pod named `secure-app` at the path `/etc/secrets`.

```shell
kubectl create ns secure
kubectl create secret generic db-credentials -n secure \
  --from-literal=username=admin \
  --from-literal=password=secretpassword123

kubectl run secure-app -n secure --image=alpine --dry-run=client -o yaml > q3.yaml
vi q3.yaml
kubectl apply -f q3.yaml
kubectl exec -n secure secure-app -- ls -la /etc/secrets
kubectl exec -n secure secure-app -- cat /etc/secrets/username
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: secure
spec:
  containers:
  - name: secure-app
    image: alpine
    command: ['sleep', '3600']
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
```

# Q4
Create a DaemonSet named `log-collector` in the `monitoring` namespace with the following specifications:
- Use the `fluentd:v1.14-1` image
- Mount the host's `/var/log` directory to `/var/log` in the container
- Add a toleration for `node-role.kubernetes.io/control-plane` with effect `NoSchedule`

```shell
kubectl create ns monitoring
vi q4.yaml
kubectl apply -f q4.yaml
kubectl get ds -n monitoring
kubectl get po -n monitoring -o wide
```

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluentd:v1.14-1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

# Q5
Create a Job named `data-processor` in the `batch` namespace that runs 3 parallel pods with the following specifications:
- Use the `busybox:1.35` image
- Each pod should run the command: `sh -c "echo Processing data batch $HOSTNAME && sleep 30"`
- Set completions to 6
- Set backoffLimit to 2

```shell
kubectl create ns batch
vi q5.yaml
kubectl apply -f q5.yaml
kubectl get jobs -n batch
kubectl get po -n batch
kubectl logs -n batch -l job-name=data-processor
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
  namespace: batch
spec:
  parallelism: 3
  completions: 6
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: processor
        image: busybox:1.35
        command:
        - sh
        - -c
        - "echo Processing data batch $HOSTNAME && sleep 30"
```

# Q6
Create a CronJob named `backup-job` in the `backup` namespace that runs every day at 2:00 AM with the following specifications:
- Use the `alpine:3.18` image
- Run the command: `sh -c "echo Backup started at $(date) && sleep 10 && echo Backup completed"`
- Set successfulJobsHistoryLimit to 3
- Set failedJobsHistoryLimit to 1

```shell
kubectl create ns backup
vi q6.yaml
kubectl apply -f q6.yaml
kubectl get cronjob -n backup
kubectl describe cronjob backup-job -n backup
```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
  namespace: backup
spec:
  schedule: "0 2 * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: alpine:3.18
            command:
            - sh
            - -c
            - "echo Backup started at $(date) && sleep 10 && echo Backup completed"
```

# Q7
Create a ServiceAccount named `api-service-account` in the `api` namespace. Create a Role named `pod-reader` that allows getting, listing, and watching pods. Bind this role to the ServiceAccount using a RoleBinding named `pod-reader-binding`.

```shell
kubectl create ns api
kubectl create serviceaccount api-service-account -n api
kubectl create role pod-reader -n api --verb=get,list,watch --resource=pods
kubectl create rolebinding pod-reader-binding -n api --role=pod-reader --serviceaccount=api:api-service-account

kubectl get sa -n api
kubectl get role -n api
kubectl get rolebinding -n api
kubectl describe rolebinding pod-reader-binding -n api
```

# Q8
Create a PersistentVolume named `shared-storage` with the following specifications:
- Storage capacity: 2Gi
- Access mode: ReadWriteMany
- Storage class: manual
- Host path: /mnt/shared-data

Then create a PersistentVolumeClaim named `shared-pvc` in the `storage` namespace that requests 1Gi of storage with ReadWriteMany access mode.

```shell
kubectl create ns storage
vi q8.yaml
kubectl apply -f q8.yaml
kubectl get pv
kubectl get pvc -n storage
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: shared-storage
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  hostPath:
    path: /mnt/shared-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
  namespace: storage
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
```

# Q9
Create a NetworkPolicy named `deny-all-ingress` in the `secure-zone` namespace that denies all ingress traffic to all pods by default. Then create another NetworkPolicy named `allow-frontend` that allows ingress traffic only from pods with the label `app=frontend` to pods with the label `app=backend` on port 8080.

```shell
kubectl create ns secure-zone
vi q9.yaml
kubectl apply -f q9.yaml
kubectl get networkpolicy -n secure-zone
kubectl describe networkpolicy deny-all-ingress -n secure-zone
kubectl describe networkpolicy allow-frontend -n secure-zone
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: secure-zone
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: secure-zone
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

# Q10
Create a multi-container Pod named `web-server` in the `web` namespace with the following specifications:
- Main container named `nginx` using `nginx:1.21` image
- Sidecar container named `log-shipper` using `busybox:1.35` image that tails the nginx access logs
- Share a volume named `logs` between containers at `/var/log/nginx` for nginx and `/shared/logs` for log-shipper
- The log-shipper should run: `tail -f /shared/logs/access.log`

```shell
kubectl create ns web
vi q10.yaml
kubectl apply -f q10.yaml
kubectl get po -n web
kubectl logs -n web web-server -c log-shipper
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
  namespace: web
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  - name: log-shipper
    image: busybox:1.35
    command:
    - sh
    - -c
    - "touch /shared/logs/access.log && tail -f /shared/logs/access.log"
    volumeMounts:
    - name: logs
      mountPath: /shared/logs
  volumes:
  - name: logs
    emptyDir: {}
```

# Q11
Create a Deployment named `web-app` in the `production` namespace with 3 replicas using the `nginx:1.21` image. Configure a readiness probe that checks HTTP GET on path `/health` at port 80 with an initial delay of 10 seconds and period of 5 seconds. Also configure a liveness probe that checks the same endpoint with an initial delay of 30 seconds and period of 10 seconds.

```shell
kubectl create deploy web-app -n production --image=nginx:1.21 --replicas=3 --dry-run=client -o yaml > q11.yaml
vi q11.yaml
kubectl apply -f q11.yaml
kubectl get deploy -n production
kubectl describe po -n production -l app=web-app
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
```

# Q12
A Pod named `broken-app` in the `debug` namespace is failing to start. Identify the issue and fix it. The Pod should use the `nginx:1.21` image and should be able to start successfully.

```shell
kubectl get po -n debug
kubectl describe po broken-app -n debug
kubectl get po broken-app -n debug -o yaml > q12.yaml
vi q12.yaml
kubectl delete po broken-app -n debug
kubectl apply -f q12.yaml
kubectl get po -n debug
```

```yaml
# First, create the broken pod for testing
apiVersion: v1
kind: Pod
metadata:
  name: broken-app
  namespace: debug
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config  # This ConfigMap doesn't exist - this is the issue
```

```yaml
# Fixed version - remove the problematic volume mount
apiVersion: v1
kind: Pod
metadata:
  name: broken-app
  namespace: debug
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```