# 7. Deploy Recovery & Data Consistency
A MariaDB Deployment in the mariadb namespace has been deleted by mistake. Your task is to restore the Deployment ensuring data persistence. Follow the steps:

Create a PersistentVolumeClaim (PVC) named mariadb in the mariadb namespace with the following specifications:
Storage 250Mi

Note - You must use the existing retained PersistentVolume (PV). There is only one existing PersistentVolume.
Edit the MariaDB Deployment file located at ~/mariadb-deployment.yaml to use the PVC you created in the previous step.
Apply the updated Deployment file to the cluster.

```bash
kubectl create ns mariadb
vi mariadb-pv.yaml
kubectl apply -f mariadb-pv.yaml

vi mariadb.yaml
kubectl apply -f mariadb.yaml
kubectl get pv,pvc

vi mariadb-deployment.yaml
k apply -f mariadb-deployment.yaml
k get deploy -n mariadb
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mariadb-pv
spec:
  capacity:
    storage: 300Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /tmp/mariadb-data
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb
  namespace: mariadb
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 250Mi
  storageClassName: standard
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
  namespace: mariadb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
        - name: mariadb
          image: mariadb:latest
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "password"
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mariadb-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mariadb-storage
          persistentVolumeClaim:
            claimName: mariadb

```