# CKA 2025 KillerCoda 실습 문제집

## 🎯 KillerCoda Playground 실습 가이드

**사용법:**
1. **KillerCoda Kubernetes Playground** 접속: https://killercoda.com/playgrounds/scenario/kubernetes
2. 각 문제의 **환경 설정** 명령어를 실행하여 문제 상황 생성
3. **문제**를 읽고 해결 시도
4. **솔루션**을 참고하여 검증

---

## 🔧 Troubleshooting 문제

### 문제 1: Pod 시작 실패 문제 해결

#### 📋 문제
Namespace `production`에 여러 Pod들이 `ImagePullBackOff`와 `CrashLoopBackOff` 오류로 시작하지 못하고 있습니다. 모든 Pod를 정상 상태로 복구하세요.

**요구사항:**
1. 실패한 Pod들과 오류 원인 식별
2. 이미지 풀 문제 해결
3. 설정 문제로 인한 크래시 해결
4. 모든 Pod가 Running 상태인지 확인

#### 🛠️ 환경 설정 (KillerCoda에서 실행)

```bash
# Namespace 생성
kubectl create namespace production

# 잘못된 이미지 이름을 가진 Deployment 생성
kubectl create deployment broken-app1 --image=nginx:wrong-tag -n production

# 리소스 부족으로 실패할 Deployment 생성
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-hungry-app
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resource-hungry-app
  template:
    metadata:
      labels:
        app: resource-hungry-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        resources:
          requests:
            cpu: "10"  # 너무 많은 CPU 요청
            memory: "10Gi"  # 너무 많은 메모리 요청
EOF

# 존재하지 않는 ConfigMap을 참조하는 Pod 생성
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-dependent-pod
  namespace: production
spec:
  containers:
  - name: app
    image: nginx:1.21
    envFrom:
    - configMapRef:
        name: non-existent-config  # 존재하지 않는 ConfigMap
EOF

# 잘못된 명령어로 크래시하는 Pod 생성
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crashing-pod
  namespace: production
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["invalid-command"]  # 존재하지 않는 명령어
EOF

echo "문제 환경이 설정되었습니다. 약 30초 후 Pod 상태를 확인하세요."
```

#### ✅ 솔루션

```bash
# 1. 실패한 Pod 확인
kubectl get pods -n production
kubectl get events -n production --sort-by='.lastTimestamp'

# 2. 각 Pod의 상세 정보 확인
kubectl describe pod -n production

# 3. 문제 해결

# 3-1. 잘못된 이미지 태그 수정
kubectl set image deployment/broken-app1 broken-app1=nginx:1.21 -n production

# 3-2. 리소스 요청량 수정
kubectl patch deployment resource-hungry-app -n production -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"100m","memory":"128Mi"}}}]}}}}'

# 3-3. 필요한 ConfigMap 생성
kubectl create configmap non-existent-config --from-literal=key1=value1 -n production

# 3-4. 크래시하는 Pod 수정
kubectl delete pod crashing-pod -n production
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crashing-pod
  namespace: production
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sleep", "3600"]  # 올바른 명령어
EOF

# 4. 최종 확인
kubectl get pods -n production
kubectl get events -n production --sort-by='.lastTimestamp' | tail -10
```

---

### 문제 2: 노드 NotReady 문제 해결

#### 📋 문제
클러스터의 워커 노드가 `NotReady` 상태입니다. kubelet 서비스에 문제가 있는 것으로 보입니다. 노드를 `Ready` 상태로 복구하세요.

**요구사항:**
1. kubelet 실패 원인 식별
2. 문제 해결 및 노드를 Ready 상태로 복구
3. 재부팅 후에도 지속되도록 설정

#### 🛠️ 환경 설정 (KillerCoda에서 실행)

```bash
# kubelet 서비스 중지 (문제 상황 시뮬레이션)
sudo systemctl stop kubelet

# kubelet 설정 파일에 오류 추가
sudo cp /var/lib/kubelet/config.yaml /var/lib/kubelet/config.yaml.backup
sudo sed -i 's/clusterDNS:/clusterDNS_BROKEN:/' /var/lib/kubelet/config.yaml

echo "kubelet이 중지되었습니다. 약 1분 후 노드가 NotReady 상태가 됩니다."
```

#### ✅ 솔루션

```bash
# 1. 노드 상태 확인
kubectl get nodes

# 2. 노드 상세 정보 확인
kubectl describe node $(hostname)

# 3. kubelet 서비스 상태 확인
sudo systemctl status kubelet

# 4. kubelet 로그 확인
sudo journalctl -u kubelet -f --no-pager | tail -20

# 5. kubelet 설정 파일 복구
sudo cp /var/lib/kubelet/config.yaml.backup /var/lib/kubelet/config.yaml

# 6. kubelet 서비스 재시작 및 활성화
sudo systemctl start kubelet
sudo systemctl enable kubelet

# 7. 노드 상태 확인
kubectl get nodes
kubectl describe node $(hostname) | grep -i ready

# 8. 시스템 Pod 상태 확인
kubectl get pods -n kube-system -o wide | grep $(hostname)
```

---

### 문제 3: 네트워크 연결 문제 해결

#### 📋 문제
`frontend` 네임스페이스의 Pod들이 `backend` 네임스페이스의 서비스에 연결할 수 없습니다. 네트워크 정책으로 인한 문제로 보입니다. 연결을 복구하세요.

**요구사항:**
1. 네임스페이스 간 연결 테스트
2. 네트워크 정책 제한 사항 식별
3. 연결 문제 해결
4. 종단 간 통신 검증

#### 🛠️ 환경 설정 (KillerCoda에서 실행)

```bash
# 네임스페이스 생성
kubectl create namespace frontend
kubectl create namespace backend

# Backend 서비스 생성
kubectl create deployment backend-app --image=nginx:1.21 -n backend
kubectl expose deployment backend-app --name=backend-service --port=80 -n backend

# Frontend Pod 생성
kubectl run frontend-pod --image=busybox:1.35 -n frontend -- sleep 3600

# 모든 트래픽을 차단하는 NetworkPolicy 생성
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# 잘못된 NetworkPolicy 생성 (잘못된 라벨 선택자)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-wrong
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: wrong-frontend  # 잘못된 라벨
    ports:
    - protocol: TCP
      port: 80
EOF

echo "네트워크 정책이 설정되었습니다. frontend에서 backend로의 연결이 차단됩니다."
```

#### ✅ 솔루션

```bash
# 1. 연결 테스트
kubectl exec -it frontend-pod -n frontend -- wget -qO- --timeout=5 http://backend-service.backend.svc.cluster.local || echo "Connection failed"

# 2. 네트워크 정책 확인
kubectl get networkpolicy -A
kubectl describe networkpolicy -n backend

# 3. 서비스 및 엔드포인트 확인
kubectl get svc -n backend
kubectl get endpoints -n backend

# 4. 네임스페이스 라벨 확인
kubectl get namespace --show-labels

# 5. 네임스페이스에 올바른 라벨 추가
kubectl label namespace frontend name=frontend

# 6. 올바른 NetworkPolicy 생성
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-correct
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 80
EOF

# 7. 잘못된 NetworkPolicy 삭제
kubectl delete networkpolicy allow-frontend-wrong -n backend

# 8. 연결 테스트 재시도
kubectl exec -it frontend-pod -n frontend -- wget -qO- --timeout=5 http://backend-service.backend.svc.cluster.local

# 9. DNS 해결 테스트
kubectl exec -it frontend-pod -n frontend -- nslookup backend-service.backend.svc.cluster.local

# 10. 최종 검증
kubectl get networkpolicy -n backend
kubectl describe networkpolicy allow-frontend-correct -n backend
```

---

## 🌐 Services and Networking 문제

### 문제 4: 복잡한 NetworkPolicy 구현

#### 📋 문제
3계층 애플리케이션 보안 모델을 NetworkPolicy로 구현하세요.

**요구사항:**
- `web-tier`: 외부에서 80포트로만 접근 가능
- `app-tier`: `web-tier`에서 8080포트로만 접근 가능  
- `db-tier`: `app-tier`에서 5432포트로만 접근 가능
- 다른 모든 통신은 차단

#### 🛠️ 환경 설정 (KillerCoda에서 실행)

```bash
# 네임스페이스 생성 및 라벨링
kubectl create namespace web-tier
kubectl create namespace app-tier
kubectl create namespace db-tier

kubectl label namespace web-tier name=web-tier
kubectl label namespace app-tier name=app-tier
kubectl label namespace db-tier name=db-tier

# 각 계층에 애플리케이션 배포
kubectl create deployment web-app --image=nginx:1.21 -n web-tier
kubectl create deployment app-server --image=nginx:1.21 -n app-tier
kubectl create deployment database --image=postgres:13 -n db-tier --env="POSTGRES_PASSWORD=secret"

# 서비스 생성
kubectl expose deployment web-app --name=web-service --port=80 -n web-tier
kubectl expose deployment app-server --name=app-service --port=8080 -n app-tier
kubectl expose deployment database --name=db-service --port=5432 -n db-tier

# 테스트용 Pod 생성
kubectl run test-pod --image=busybox:1.35 -n web-tier -- sleep 3600

echo "3계층 애플리케이션이 배포되었습니다. 현재는 모든 통신이 허용됩니다."
```

#### ✅ 솔루션

```bash
# 1. 현재 연결 상태 테스트 (모든 연결이 가능해야 함)
kubectl exec -it test-pod -n web-tier -- wget -qO- --timeout=5 http://app-service.app-tier.svc.cluster.local:8080 || echo "Connection failed"

# 2. Web-tier NetworkPolicy 생성
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-tier-policy
  namespace: web-tier
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: app-tier
    ports:
    - protocol: TCP
      port: 8080
  - to: []  # DNS 허용
    ports:
    - protocol: UDP
      port: 53
EOF

# 3. App-tier NetworkPolicy 생성
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-tier-policy
  namespace: app-tier
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: web-tier
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: db-tier
    ports:
    - protocol: TCP
      port: 5432
  - to: []  # DNS 허용
    ports:
    - protocol: UDP
      port: 53
EOF

# 4. DB-tier NetworkPolicy 생성
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-tier-policy
  namespace: db-tier
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: app-tier
    ports:
    - protocol: TCP
      port: 5432
EOF

# 5. NetworkPolicy 확인
kubectl get networkpolicy -A

# 6. 허용된 연결 테스트
kubectl run test-web -n web-tier --image=busybox:1.35 --rm -it -- wget -qO- --timeout=5 http://app-service.app-tier.svc.cluster.local:8080

# 7. 차단된 연결 테스트 (실패해야 함)
kubectl run test-direct -n web-tier --image=busybox:1.35 --rm -it -- wget -qO- --timeout=5 http://db-service.db-tier.svc.cluster.local:5432 || echo "Connection blocked (expected)"

# 8. 정책 상세 확인
kubectl describe networkpolicy -n web-tier
kubectl describe networkpolicy -n app-tier
kubectl describe networkpolicy -n db-tier
```

---

### 문제 5: Ingress와 TLS 설정

#### 📋 문제
여러 도메인에 대한 TLS 종료를 처리하는 Ingress 컨트롤러를 구성하세요.

**요구사항:**
- `api.example.com` → `api-service:8080`
- `web.example.com` → `web-service:80`
- 두 도메인 모두 TLS 사용
- HTTP를 HTTPS로 리다이렉트

#### 🛠️ 환경 설정 (KillerCoda에서 실행)

```bash
# 네임스페이스 생성
kubectl create namespace web-services

# 애플리케이션 배포
kubectl create deployment api-app --image=nginx:1.21 -n web-services
kubectl create deployment web-app --image=nginx:1.21 -n web-services

# 서비스 생성
kubectl expose deployment api-app --name=api-service --port=8080 --target-port=80 -n web-services
kubectl expose deployment web-app --name=web-service --port=80 -n web-services

# NGINX Ingress Controller 설치 (KillerCoda에서는 이미 설치되어 있을 수 있음)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

echo "애플리케이션이 배포되었습니다. TLS 인증서와 Ingress를 설정하세요."
```

#### ✅ 솔루션

```bash
# 1. TLS 인증서 생성
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout api-tls.key -out api-tls.crt \
  -subj "/CN=api.example.com"

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout web-tls.key -out web-tls.crt \
  -subj "/CN=web.example.com"

# 2. TLS Secret 생성
kubectl create secret tls api-tls-secret \
  --cert=api-tls.crt --key=api-tls.key -n web-services

kubectl create secret tls web-tls-secret \
  --cert=web-tls.crt --key=web-tls.key -n web-services

# 3. Ingress 생성
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-domain-ingress
  namespace: web-services
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-secret
  - hosts:
    - web.example.com
    secretName: web-tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

# 4. Ingress 상태 확인
kubectl get ingress -n web-services
kubectl describe ingress multi-domain-ingress -n web-services

# 5. NGINX Ingress Controller 상태 확인
kubectl get pods -n ingress-nginx

# 6. 로컬 테스트를 위한 hosts 파일 설정 (선택사항)
echo "127.0.0.1 api.example.com web.example.com" | sudo tee -a /etc/hosts

# 7. 테스트 (Ingress Controller의 외부 IP가 있는 경우)
# curl -k https://api.example.com
# curl -k https://web.example.com

# 8. Secret 확인
kubectl get secrets -n web-services
kubectl describe secret api-tls-secret -n web-services
```

---

## ⚙️ Cluster Architecture 문제

### 문제 6: ETCD 백업 및 복원

#### 📋 문제
ETCD 데이터베이스의 백업을 생성하고, 새로운 데이터 디렉토리로 복원하세요.

**요구사항:**
1. 현재 ETCD 데이터의 완전한 백업 생성
2. 백업을 새 데이터 디렉토리로 복원
3. 클러스터 상태 및 데이터 무결성 검증

#### 🛠️ 환경 설정 (KillerCoda에서 실행)

```bash
# 테스트용 데이터 생성
kubectl create namespace backup-test
kubectl create configmap test-data --from-literal=key1=value1 --from-literal=key2=value2 -n backup-test
kubectl create secret generic test-secret --from-literal=username=admin --from-literal=password=secret123 -n backup-test

# 현재 데이터 확인
kubectl get all -n backup-test
kubectl get configmap test-data -n backup-test -o yaml
kubectl get secret test-secret -n backup-test -o yaml

echo "테스트 데이터가 생성되었습니다. ETCD 백업을 진행하세요."
```

#### ✅ 솔루션

```bash
# 1. ETCD Pod 및 인증서 위치 확인
kubectl get pods -n kube-system | grep etcd
sudo find /etc/kubernetes -name "*.crt" -o -name "*.key" | grep etcd

# 2. ETCD 백업 생성
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup-$(date +%Y%m%d-%H%M%S).db

# 3. 백업 파일 확인
ls -la /opt/etcd-backup-*.db

# 4. 백업 상태 검증
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup-*.db

# 5. 복원을 위한 새 데이터 디렉토리 생성
sudo mkdir -p /var/lib/etcd-restore
sudo chown -R etcd:etcd /var/lib/etcd-restore

# 6. 백업에서 복원
ETCDCTL_API=3 etcdctl --data-dir=/var/lib/etcd-restore \
  snapshot restore /opt/etcd-backup-*.db

# 7. etcd.yaml 백업
sudo cp /etc/kubernetes/manifests/etcd.yaml /etc/kubernetes/manifests/etcd.yaml.backup

# 8. etcd.yaml 수정 (데이터 디렉토리 변경)
sudo sed -i 's|/var/lib/etcd|/var/lib/etcd-restore|g' /etc/kubernetes/manifests/etcd.yaml

# 9. etcd Pod 재시작 대기
kubectl get pods -n kube-system | grep etcd

# 10. 클러스터 상태 확인
kubectl get nodes
kubectl get pods -A

# 11. 복원된 데이터 확인
kubectl get configmap test-data -n backup-test -o yaml
kubectl get secret test-secret -n backup-test -o yaml

# 12. ETCD 클러스터 상태 확인
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

echo "ETCD 백업 및 복원이 완료되었습니다."
```

---

### 문제 7: 클러스터 업그레이드

#### 📋 문제
Kubernetes 클러스터를 현재 버전에서 다음 마이너 버전으로 업그레이드하세요.

**요구사항:**
1. Control Plane 노드 업그레이드
2. 클러스터 기능 검증
3. 모든 시스템 Pod가 정상 실행 중인지 확인

#### 🛠️ 환경 설정 (KillerCoda에서 실행)

```bash
# 현재 클러스터 버전 확인
kubectl version --short
kubectl get nodes

# 업그레이드 전 상태 확인
kubectl get pods -A
kubectl get componentstatuses 2>/dev/null || echo "ComponentStatus API deprecated"

# 테스트 워크로드 생성
kubectl create deployment test-app --image=nginx:1.21 --replicas=2
kubectl expose deployment test-app --port=80 --type=ClusterIP

echo "클러스터 업그레이드를 시작하세요. 현재 버전: $(kubectl version --short)"
```

#### ✅ 솔루션

```bash
# 1. 현재 버전 확인
kubectl version --short
kubeadm version

# 2. 사용 가능한 kubeadm 버전 확인
sudo apt update
sudo apt-cache madison kubeadm | head -5

# 3. kubeadm 업그레이드 (예: 1.28.x -> 1.29.x)
# 실제 환경에서는 적절한 버전을 선택하세요
UPGRADE_VERSION="1.29.0-1.1"

sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm=$UPGRADE_VERSION && \
sudo apt-mark hold kubeadm

# 4. kubeadm 버전 확인
kubeadm version

# 5. 업그레이드 계획 확인
sudo kubeadm upgrade plan

# 6. 업그레이드 적용 (첫 번째 control plane 노드에서)
sudo kubeadm upgrade apply v1.29.0

# 7. 노드 드레인 (단일 노드 클러스터에서는 주의)
kubectl drain $(hostname) --ignore-daemonsets --delete-emptydir-data

# 8. kubelet과 kubectl 업그레이드
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet=$UPGRADE_VERSION kubectl=$UPGRADE_VERSION && \
sudo apt-mark hold kubelet kubectl

# 9. kubelet 재시작
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 10. 노드 언코든
kubectl uncordon $(hostname)

# 11. 업그레이드 확인
kubectl version --short
kubectl get nodes
kubectl get pods -A

# 12. 테스트 워크로드 확인
kubectl get deployment test-app
kubectl get pods -l app=test-app

# 13. 클러스터 상태 최종 확인
kubectl cluster-info
kubectl get events --sort-by='.lastTimestamp' | tail -10

echo "클러스터 업그레이드가 완료되었습니다."
```

---

## 📦 Workloads & Scheduling 문제

### 문제 8: 고급 스케줄링 및 Affinity

#### 📋 문제
복잡한 Pod 스케줄링 요구사항을 구현하세요.

**요구사항:**
- 데이터베이스 Pod는 SSD 스토리지가 있는 노드에만 스케줄
- 데이터베이스 Pod들은 서로 다른 가용 영역에 분산
- 웹과 캐시 Pod는 같은 노드에 배치
- 유지보수 중인 노드에 대한 적절한 toleration 설정

#### 🛠️ 환경 설정 (KillerCoda에서 실행)

```bash
# 네임스페이스 생성
kubectl create namespace scheduling-test

# 노드 라벨링 (가상의 스토리지 타입과 가용 영역)
kubectl label node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') storage-type=ssd
kubectl label node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') topology.kubernetes.io/zone=zone-a

# 추가 노드가 있다면 다른 라벨 설정 (KillerCoda는 보통 단일 노드)
# kubectl label node worker-node-02 storage-type=hdd
# kubectl label node worker-node-02 topology.kubernetes.io/zone=zone-b

# 유지보수 테인트 설정
kubectl taint node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') maintenance=true:NoSchedule

echo "노드 라벨링과 테인트가 설정되었습니다."
kubectl get nodes --show-labels
```

#### ✅ 솔루션

```bash
# 1. 노드 상태 확인
kubectl get nodes --show-labels
kubectl describe nodes | grep -i taint

# 2. 데이터베이스 Deployment (Node Affinity + Pod Anti-Affinity + Toleration)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: scheduling-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: storage-type
                operator: In
                values:
                - ssd
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - database
              topologyKey: topology.kubernetes.io/zone
      tolerations:
      - key: maintenance
        operator: Equal
        value: "true"
        effect: NoSchedule
        tolerationSeconds: 300
      containers:
      - name: database
        image: postgres:13
        env:
        - name: POSTGRES_PASSWORD
          value: "secret"
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
EOF

# 3. 캐시 Deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache
  namespace: scheduling-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cache
  template:
    metadata:
      labels:
        app: cache
    spec:
      tolerations:
      - key: maintenance
        operator: Equal
        value: "true"
        effect: NoSchedule
      containers:
      - name: cache
        image: redis:6-alpine
        resources:
          requests:
            cpu: 50m
            memory: 128Mi
EOF

# 4. 웹 애플리케이션 Deployment (Pod Affinity로 캐시와 같은 노드에 배치)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: scheduling-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - cache
              topologyKey: kubernetes.io/hostname
      tolerations:
      - key: maintenance
        operator: Equal
        value: "true"
        effect: NoSchedule
      containers:
      - name: web
        image: nginx:1.21
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
EOF

# 5. Pod 배치 확인
kubectl get pods -n scheduling-test -o wide

# 6. Pod 스케줄링 상세 정보 확인
kubectl describe pod -n scheduling-test | grep -A 10 "Node-Selectors\|Tolerations\|Events"

# 7. 노드별 Pod 분산 확인
kubectl get pods -n scheduling-test -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase

# 8. 테인트 제거 테스트 (선택사항)
kubectl taint node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') maintenance-

echo "고급 스케줄링 설정이 완료되었습니다."
```

---

### 문제 9: HPA와 리소스 관리

#### 📋 문제
포괄적인 리소스 관리를 구성하세요. HPA, 리소스 쿼터, 리밋 레인지를 포함합니다.

**요구사항:**
1. CPU와 메모리 기반 HPA 설정
2. 네임스페이스 리소스 쿼터 구현
3. 컨테이너 리소스 제한 설정
4. 리소스 사용량 모니터링 및 최적화

#### 🛠️ 환경 설정 (KillerCoda에서 실행)

```bash
# 네임스페이스 생성
kubectl create namespace resource-management

# Metrics Server 설치 (HPA를 위해 필요)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Metrics Server가 KillerCoda에서 작동하도록 설정 수정
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

echo "Metrics Server가 설치되었습니다. 잠시 후 메트릭을 사용할 수 있습니다."
```

#### ✅ 솔루션

```bash
# 1. Metrics Server 상태 확인
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes 2>/dev/null || echo "Metrics not ready yet, waiting..."

# 2. Resource Quota 생성
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-quota
  namespace: resource-management
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    persistentvolumeclaims: "2"
    pods: "5"
    services: "3"
EOF

# 3. Limit Range 생성
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
  namespace: resource-management
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
  - max:
      cpu: 1
      memory: 1Gi
    min:
      cpu: 50m
      memory: 64Mi
    type: Container
EOF

# 4. 애플리케이션 Deployment 생성
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: resource-management
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: nginx:1.21
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        ports:
        - containerPort: 80
EOF

# 5. 서비스 생성
kubectl expose deployment web-app --name=web-app-service --port=80 -n resource-management

# 6. HPA 생성 (CPU와 메모리 기반)
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: resource-management
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
EOF

# 7. 리소스 상태 확인
kubectl get resourcequota -n resource-management
kubectl describe resourcequota resource-quota -n resource-management

kubectl get limitrange -n resource-management
kubectl describe limitrange limit-range -n resource-management

# 8. HPA 상태 확인
kubectl get hpa -n resource-management
kubectl describe hpa web-app-hpa -n resource-management

# 9. Pod 리소스 사용량 확인 (메트릭이 준비되면)
sleep 30
kubectl top pods -n resource-management

# 10. 부하 테스트 (CPU 사용량 증가)
kubectl run load-generator --image=busybox:1.35 -n resource-management --rm -it --restart=Never -- \
  sh -c "while true; do wget -q -O- http://web-app-service.resource-management.svc.cluster.local; done" &

# 11. HPA 동작 모니터링
kubectl get hpa -n resource-management -w &
sleep 60
kubectl get pods -n resource-management

# 12. 부하 테스트 중지
pkill -f "wget.*web-app-service"

echo "리소스 관리 설정이 완료되었습니다."
```

---

## 💾 Storage 문제

### 문제 10: 동적 스토리지 프로비저닝

#### 📋 문제
다양한 성능 계층의 스토리지 클래스를 생성하고 볼륨 확장 기능을 구현하세요.

**요구사항:**
1. 빠른 SSD와 표준 HDD용 스토리지 클래스 생성
2. 볼륨 확장 기능 활성화
3. StatefulSet에서 동적 스토리지 사용
4. 볼륨 확장 테스트

#### 🛠️ 환경 설정 (KillerCoda에서 실행)

```bash
# 네임스페이스 생성
kubectl create namespace storage-demo

# 기본 스토리지 클래스 확인
kubectl get storageclass

echo "스토리지 클래스를 생성하고 동적 프로비저닝을 설정하세요."
```

#### ✅ 솔루션

```bash
# 1. 현재 스토리지 클래스 확인
kubectl get storageclass

# 2. Fast SSD Storage Class 생성
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: k8s.io/minikube-hostpath  # KillerCoda/Minikube용
parameters:
  type: "fast"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
EOF

# 3. Standard HDD Storage Class 생성
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-hdd
provisioner: k8s.io/minikube-hostpath  # KillerCoda/Minikube용
parameters:
  type: "standard"
allowVolumeExpansion: true
volumeBindingMode: Immediate
reclaimPolicy: Retain
EOF

# 4. StatefulSet with 동적 스토리지 생성
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
  namespace: storage-demo
spec:
  serviceName: database
  replicas: 2
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_PASSWORD
          value: "secret"
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 1Gi
EOF

# 5. Headless Service 생성
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: storage-demo
spec:
  clusterIP: None
  selector:
    app: database
  ports:
  - port: 5432
EOF

# 6. 별도 PVC 생성 (백업용)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backup-pvc
  namespace: storage-demo
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard-hdd
  resources:
    requests:
      storage: 2Gi
EOF

# 7. 스토리지 상태 확인
kubectl get storageclass
kubectl get pv
kubectl get pvc -n storage-demo

# 8. StatefulSet 상태 확인
kubectl get statefulset -n storage-demo
kubectl get pods -n storage-demo

# 9. 볼륨 확장 테스트
kubectl patch pvc data-database-0 -n storage-demo -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'

# 10. 확장 상태 확인
kubectl describe pvc data-database-0 -n storage-demo
kubectl get pvc -n storage-demo

# 11. Pod 내에서 디스크 공간 확인
kubectl exec -it database-0 -n storage-demo -- df -h /var/lib/postgresql/data

# 12. 데이터 지속성 테스트
kubectl exec -it database-0 -n storage-demo -- psql -U postgres -c "CREATE TABLE test_table (id SERIAL PRIMARY KEY, data TEXT);"
kubectl exec -it database-0 -n storage-demo -- psql -U postgres -c "INSERT INTO test_table (data) VALUES ('persistent data');"

# 13. Pod 재시작 후 데이터 확인
kubectl delete pod database-0 -n storage-demo
kubectl wait --for=condition=Ready pod/database-0 -n storage-demo --timeout=60s
kubectl exec -it database-0 -n storage-demo -- psql -U postgres -c "SELECT * FROM test_table;"

echo "동적 스토리지 프로비저닝 설정이 완료되었습니다."
```

---

## 🔐 Security 문제

### 문제 11: 종합 보안 구성

#### 📋 문제
RBAC, Pod Security Standards, 네트워크 정책을 포함한 종합적인 보안 조치를 구현하세요.

**요구사항:**
1. 사용자 정의 RBAC 역할 및 바인딩 생성
2. Pod Security Standards 구현
3. 최소 권한 원칙으로 서비스 계정 구성
4. 보안 정책 및 네트워크 정책 설정

#### 🛠️ 환경 설정 (KillerCoda에서 실행)

```bash
# 보안 네임스페이스 생성
kubectl create namespace secure-apps

# 테스트용 비보안 Pod 생성 (나중에 보안 정책으로 차단될 예정)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
  namespace: secure-apps
spec:
  containers:
  - name: app
    image: nginx:1.21
    securityContext:
      privileged: true  # 보안 정책에 의해 차단될 예정
EOF

echo "보안 환경 설정이 완료되었습니다. 보안 정책을 구현하세요."
```

#### ✅ 솔루션

```bash
# 1. 기존 비보안 Pod 삭제
kubectl delete pod insecure-pod -n secure-apps --force --grace-period=0

# 2. Pod Security Standards 적용
kubectl label namespace secure-apps \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest

# 3. 사용자 정의 ClusterRole 생성
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-manager
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
EOF

# 4. ServiceAccount 생성
kubectl create serviceaccount pod-manager-sa -n secure-apps

# 5. ClusterRoleBinding 생성
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-manager-binding
subjects:
- kind: ServiceAccount
  name: pod-manager-sa
  namespace: secure-apps
roleRef:
  kind: ClusterRole
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
EOF

# 6. 보안 Pod 생성 (Restricted 정책 준수)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: secure-apps
  labels:
    app: secure-app
spec:
  serviceAccountName: pod-manager-sa
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:1.21
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: var-cache
      mountPath: /var/cache/nginx
    - name: var-run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: var-cache
    emptyDir: {}
  - name: var-run
    emptyDir: {}
EOF

# 7. 네트워크 정책 생성
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: secure-app-netpol
  namespace: secure-apps
spec:
  podSelector:
    matchLabels:
      app: secure-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: client
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to: []  # DNS 허용
    ports:
    - protocol: UDP
      port: 53
  - to:
    - podSelector:
        matchLabels:
          role: database
    ports:
    - protocol: TCP
      port: 5432
EOF

# 8. RBAC 테스트
kubectl auth can-i create pods --as=system:serviceaccount:secure-apps:pod-manager-sa
kubectl auth can-i delete nodes --as=system:serviceaccount:secure-apps:pod-manager-sa
kubectl auth can-i get pods --as=system:serviceaccount:secure-apps:pod-manager-sa -n secure-apps

# 9. Pod Security Standards 테스트 (실패해야 함)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: insecure-test-pod
  namespace: secure-apps
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: true  # 이것은 거부되어야 함
EOF

# 10. 보안 Pod 상태 확인
kubectl get pods -n secure-apps
kubectl describe pod secure-app -n secure-apps

# 11. 네트워크 정책 확인
kubectl get networkpolicy -n secure-apps
kubectl describe networkpolicy secure-app-netpol -n secure-apps

# 12. 보안 컨텍스트 확인
kubectl get pod secure-app -n secure-apps -o jsonpath='{.spec.securityContext}' | jq .
kubectl get pod secure-app -n secure-apps -o jsonpath='{.spec.containers[0].securityContext}' | jq .

# 13. 서비스 계정 토큰 확인
kubectl describe serviceaccount pod-manager-sa -n secure-apps

echo "종합 보안 구성이 완료되었습니다."
```

---

## 📝 KillerCoda 실습 가이드

### 🎯 효과적인 실습 방법

1. **순서대로 실습**: 문제 1부터 차례대로 진행
2. **환경 초기화**: 각 문제 시작 전 `kubectl delete namespace <namespace>` 로 정리
3. **시간 측정**: 실제 시험처럼 시간을 재며 연습
4. **문서 참조**: Kubernetes 공식 문서를 적극 활용

### 🔧 KillerCoda 환경 최적화

```bash
# kubectl 자동완성 설정
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# 유용한 alias 설정
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'

# 환경변수 설정
export do="--dry-run=client -o yaml"
export now="--force --grace-period 0"
```

### 📚 추가 학습 리소스

- **KillerCoda Kubernetes Scenarios**: https://killercoda.com/kubernetes
- **Kubernetes 공식 문서**: https://kubernetes.io/docs/
- **kubectl 치트시트**: https://kubernetes.io/docs/reference/kubectl/cheatsheet/

---

**KillerCoda에서 실습하며 CKA 시험을 완벽하게 준비하세요! 🚀**

**Good Luck! 💪**