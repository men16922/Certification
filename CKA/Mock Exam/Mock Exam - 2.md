# Q1
Create a StorageClass named local-sc with the following specifications and set it as the default storage class:

The provisioner should be kubernetes.io/no-provisioner
The volume binding mode should be WaitForFirstConsumer
Volume expansion should be enabled

Is the StorageClass local-sc created?

Is Provisioner kubernetes.io/no-provisioner used?

Is the volume binding set to WaitForFirstConsumer?

Is local-sc set to the default storage class?

```shell
vi q1.yaml
kubectl apply -f q1.yaml
kubectl get sc
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/no-provisioner
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
``

# Q2
Create a deployment named logging-deployment in the namespace logging-ns with 1 replica, with the following specifications:

The main container should be named app-container, use the image busybox, and should run the following command to simulate writing logs:

sh -c "while true; do echo 'Log entry' >> /var/log/app/app.log; sleep 5; done"

Add a sidecar container named log-agent that also uses the busybox image and runs the command:

tail -f /var/log/app/app.log

log-agent logs should display the entries logged by the main app-container

```shell
kubectl create deploy logging-deployment -n logging-ns --image=busybox --dry-run=client -o yaml > q2.yaml
vi q2.yaml
kubectl apply -f q2.yaml
kubectl get deploy -n logging-ns

kubectl logs -n logging-ns deployment/logging-deployment -c log-agent
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logging-deployment
  namespace: logging-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: logger
  template:
    metadata:
      labels:
        app: logger
    spec:
      volumes:
        - name: log-volume
          emptyDir: {}
      initContainers:
        - name: log-agent
          image: busybox
          command:
            - sh
            - -c
            - "touch /var/log/app/app.log; tail -f /var/log/app/app.log"
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
          restartPolicy: Always 
      containers:
        - name: app-container
          image: busybox
          command:
            - sh
            - -c
            - "while true; do echo 'Log entry' >> /var/log/app/app.log; sleep 5; done"
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
```


# Q3
A Deployment named webapp-deploy is running in the ingress-ns namespace and is exposed via a Service named webapp-svc.

Create an Ingress resource called webapp-ingress in the same namespace that will route traffic to the service. The Ingress must:

Use pathType: Prefix
Route requests sent to path / to the backend service
Forward traffic to port 80 of the service
Be configured for the host kodekloud-ingress.app
Test app availablility using the following command:

curl -s http://kodekloud-ingress.app/


Ingress exposed and serving traffic via kodekloud-ingress.app host

```shell
vi q3.yaml
kubectl apply -f q3.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: ingress-ns
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: "kodekloud-ingress.app"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-svc
            port:
              number: 80
```

# Q4
Create a new deployment called nginx-deploy, with image nginx:1.16 and 1 replica. Next, upgrade the deployment to version 1.17 using rolling update.


Deployment: nginx-deploy, Image: nginx:1.16

Image: nginx:1.16

Version upgraded to 1.17

```shell
kubectl create deploy nginx-deploy --image=nginx:1.16 --replicas=1
kubectl set image deploy/nginx-deploy nginx=nginx:1.17
kubectl rollout status deploy/nginx-deploy
kubectl rollout history deploy/nginx-deploy
```

# Q5
Create a new user called john. Grant him access to the cluster using a csr named john-developer. Create a role developer which should grant John the permission to create, list, get, update and delete pods in the development namespace . The private key exists in the location: /root/CKA/john.key and csr at /root/CKA/john.csr.


Important Note: As of kubernetes 1.19, the CertificateSigningRequest object expects a signerName.

Please refer to the documentation to see an example. The documentation tab is available at the top right of the terminal.

CSR: john-developer Status:Approved

Role Name: developer, namespace: development, Resource: Pods

Access: User 'john' has appropriate permissions

```shell
cat /root/CKA/john.csr | base64 | tr -d '\n'
kubectl create sa john
kubectl apply -f john.yaml
kubectl get csr
kubectl certificate approve john-developer
kubectl get csr/john-developer -o yaml
kubectl get csr john-developer -o jsonpath='{.status.certificate}'| base64 -d > /root/CKA/john.crt

kubectl create role developer -n development --verb=create,list,get,update,delete --resource=pods
kubectl get role -n development developer -o yaml
kubectl create rolebinding developer-rolebinding  --user=john --namespace=development --role=developer
kubectl get rolebinding -n development developer-rolebinding -o yaml
```

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-developer
spec:
  # This is an encoded CSR. Change this to the base64-encoded contents of myuser.csr
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZEQ0NBVHdDQVFBd0R6RU5NQXNHQTFVRUF3d0VhbTlvYmpDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRApnZ0VQQURDQ0FRb0NnZ0VCQUsrU1BFY0U0eVgzTTFoMVpCTW5uY1hIY3B6TExuOGtQZm9pTmIzaEoyVFpkY1VXCklmYng5TitYSDZRU2ZqSWVVUTV6dFdZcEJIZDNzemNtcjlXTVJFMkxDS1BpUzZQYW5ENXN5c0RCMngzQ2t1Q3kKa0R1VVNYNjFwdzUxU2dmY082N1poM3NqTDVQbTRBWUZyM0VLc1k3NVllUWFLcUhXNjRZWmZRVDZiV1hmb1RSUwpJaWdweTNFTlo5SG9aTXZ6QW5rdVNBQzVMdURRbVBSSkRsUmN2ZDArUnVQVzV6NjBEMDE3VDRuOENPRmx2M0kwCjAxQ1p3MFZuODROcTFRZy82Uy8yN1dxWFpYSE4vZkZvQWFNWTZ6TWpSdGV0VnUvN1VGMWxZZFV5T09CR2Z4VXYKNTZ3bGJvV001OFowa0hQdDBjWDVhVXpSYWc4TGZpNE14WGJIUGtFQ0F3RUFBYUFBTUEwR0NTcUdTSWIzRFFFQgpDd1VBQTRJQkFRQ0dLZGxCNDAxTlVqQTZMTXRZR3lVMkdwVStnSjdiT0I2Ri9lVWxLSkNEUUxKellmRFpoeWRqClErbjVESzdWNHU4cEk1NkxUMWowWjF1QVdiT2lmcWJhdlZLMFVhRVRSQ1BUbmxGK1M4cGdIazErSlM0dFVLRzEKenNQbS9rVWVtT0NDZkNhWVhUWUs4N0JMcnNVYStsWE9HSjVaM0IyNDR4dmFGQWVZcXh4amNmMGVsR0hHMWdTNQptVUdVZ1Rpak1YWFJsdHY5dWN4dERZZHZlQnFlVGpYQmxvK3czS1JJdFp2bUdidjZNMVlPczQzMVk2eFdzVUxWCnR3c1FtckEyOUMwZjdzR3pzL01XT2RRSklnYXBkNGZPby9NRGxQOSthVUtwRHlvNjNhSno5NnVnNWRuVTRXZTAKYm04aFB3ZjlCS3MvWTBNOHhKaWRBMEZ0VnpqUDJSdkUKLS0tLS1FTkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

# Q6
Create an nginx pod called nginx-resolver using the image nginx and expose it internally with a service called nginx-resolver-service. Test that you are able to look up the service and pod names from within the cluster. Use the image: busybox:1.28 for dns lookup. Record results in /root/CKA/nginx.svc and /root/CKA/nginx.pod


Pod: nginx-resolver created

Service DNS Resolution recorded correctly

Pod DNS resolution recorded correctly

```shell
kubectl run nginx-resolver --image=nginx
kubectl expose pod nginx-resolver --name=nginx-resolver-service --port 80
kubectl run test-nslookup --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx-resolver-service > /root/CKA/nginx.svc
kubectl run test-nslookup --image=busybox:1.28 --rm -it --restart=Never -- nslookup 172-17-0-9.default.pod.cluster.local > /root/CKA/nginx.pod


cat /root/CKA/nginx.svc
cat /root/CKA/nginx.pod

```

# Q7
Create a static pod on node01 called nginx-critical with the image nginx. Make sure that it is recreated/restarted automatically in case of a failure.


For example, use /etc/kubernetes/manifests as the static Pod path.

Is the static pod configured under /etc/kubernetes/manifests?

Is pod nginx-critical-cluster1-node01 up and running?

```shell
ssh node01
vi /etc/kubernetes/manifests/nginx-critical.yaml
k get po -o wide
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-critical
spec:
  containers:
  - name: nginx-critical
    image: nginx
    ports:
    - containerPort: 80
```

# Q8
Create a Horizontal Pod Autoscaler with name backend-hpa for the deployment named backend-deployment in the backend namespace with the webapp-hpa.yaml file located under the root folder.
Ensure that the HPA scales the deployment based on memory utilization, maintaining an average memory usage of 65% across all pods.
Configure the HPA with a minimum of 3 replicas and a maximum of 15.


Is backend-hpa HPA deployed in backend namespace?

Is deployment configured for metrics memory utilization?

```shell
vi webapp-hpa.yaml 
kubectl apply -f webapp-hpa.yaml 
kubectl get hpa -n backend
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-deployment
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 65
```

# Q9
Modify the existing web-gateway on cka5673 namespace to handle HTTPS traffic on port 443 for kodekloud.com, using a TLS certificate stored in a secret named kodekloud-tls.


Is the web gateway configured to listen on the hostname kodekloud.com?

Is the HTTPS listener configured with the correct TLS certificate?

```shell
kubectl get gateway web-gateway -n cka5673 -o yaml > q9.yaml
kubectl get secret kodekloud-tls -n cka5673 -o yaml

kubectl delete gateway web-gateway -n cka5673
vi q9.yaml
kubectl apply -f q9.yaml
kubectl get gateway web-gateway -n cka5673 -o yaml

```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: cka5673
spec:
  gatewayClassName: kodekloud
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: kodekloud.com
      tls:
        certificateRefs:
          - name: kodekloud-tls
```

# Q10
On the cluster, the team has installed multiple helm charts on a different namespace. By mistake, those deployed resources include one of the vulnerable images called kodekloud/webapp-color:v1. Find out the release name and uninstall it.


Is helm release uninstalled?

```shell
helm list -A
k get po web-dashboard-apd-5bb6644bc8-gjx4j -n web-dashboard-03 -o yaml

kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | grep kodekloud/webapp-color:v1
helm list -n atlanta-page-04
helm uninstall atlanta-page-apd -n atlanta-page-04
```

# Q11
You are requested to create a NetworkPolicy to allow traffic from frontend apps located in the frontend namespace, to backend apps located in the backend namespace, but not from the databases in the databases namespace. There are three policies available in the /root folder. Apply the most restrictive policy from the provided YAML files to achieve the desired result. Do not delete any existing policies.


Correct NetworkPolicy applied

Incorrect NetworkPolicy is not applied

Second incorrect NetworkPolicy is not applied

```shell
cat net-pol-1.yaml
cat net-pol-2.yaml
cat net-pol-3.yaml

kubectl apply -f net-pol-3.yaml
kubectl get networkpolicy -n backend
kubectl describe networkpolicy net-policy-3 -n backend
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: net-policy-3
  namespace: backend
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 80
```