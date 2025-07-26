# 2. Deploy & NodePort 2025
Question No: 2
Weightage: 10%

Reconfigure the existing deployment front-end in namespace spline-reticulator to expose port 80/tcp of the existing container nginx.

Create a new service named front-end-svc exposing the container port 80/tcp.

Configure the new service to also expose the individual pod via a NodePort.



```bash
kubectl create ns spline-reticulator
kubectl create deploy front-end --image=nginx -n spline-reticulator
kubectl edit deploy front-end -n spline-reticulator

kubectl expose deploy front-end --name=front-end-svc --port 80 --target-port 80 -n spline-reticulator
kubectl patch svc front-end-svc -p '{"spec": {"type": "NodePort"}}' -n spline-reticulator

```

```yaml
spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        **ports:
        - containerPort: 80
          protocol: TCP**
```