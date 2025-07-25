[TOC]

# 실전 문제 풀이

### ❓1. Retrieve Error Messages from a Container Log

- Cluster: kubectl config use-context **hk8s**

---

In the `customera` namespace, check the log for the nginx container in the `custom-app` Pod.

Save the lines which contain the text “error” to the file /data/CKA/errors.txt.

---

- ❗Answer

    ```bash
    kubectl config use-context hk8s
    
    **kubectl get pods  -n customera**
    kubectl logs custom-app -n customera | grep -i error > /data/CKA/errors.txt
    cat /data/CKA/errors.txt
    ```


### ❓2. Node Troubleshooting

- Cluster: kubectl config use-context **hk8s**

---

A Kubernetes worker node, named hk8s-worker2 is in state NotReady.
Investigate why this is the case, and perform any appropriate steps to bring the node to a Ready state, ensuring that any changes are made permanent.

---

- ❗Answer

    ```bash
    #
    **console$ ssh hk8s-master
    kubectl get nodes**
    NAME           STATUS     ROLES           AGE   VERSION
    hk8s-master    Ready      control-plane   70d   v1.28.0
    hk8s-worker1   Ready      <none>          70d   v1.28.0
    hk8s-worker2   **NotReady**   <none>          70d   v1.28.0
    
    **ssh hk8s-worker2** 
    sudo -i
    # containerd runtime 동작 상태 확인
    **systemctl status containerd**
    
    # kubelet 동작 상태 확인
    **systemctl status kubelet**
    **systemctl enable --now kubelet**  # enable : 부팅시 자동으로 데몬 실행되도록 설정. now 옵션 추가로 즉시로 서비스 동작
    **systemctl status kubelet**
    exit
    exit
    
    **console$ kubectl get nodes**
    NAME           STATUS     ROLES           AGE   VERSION
    hk8s-master    Ready      control-plane   70d   v1.28.0
    hk8s-worker1   Ready      <none>          70d   v1.28.0
    hk8s-worker2   Ready      <none>          70d   v1.28.0
    ```

    <aside>
    📌 Node Troubleshooting 문제 해결을 하면 hk8s-worker1, hk8s-worker2가 모두 Ready 상태가 됩니다. 이후 다음 작업 진행해주세요.
    1. hk8s-master, hk8s-worker1, hk8s-worker2 시스템을 모두 종료하세요. 
    2. hk8s-worker1 서버의 메모리를 2G로 변경
    3. hk8s-master, hk8s-worker1, hk8s-worker2 시스템을 모두 부팅

    </aside>


### ❓3. Count the Number of Nodes That Are Ready to Run Normal Workloads

- Cluster: kubectl config use-context **hk8s**

---

Determine how many nodes in the cluster are ready to run **normal workloads** (i.e., workloads that **do not have any special tolerations**).

Output this number to the file  /var/CKA2022/count.txt

---

- ❗Answer

    ```bash
    # 클러스터에서 normal workload를 실행할 ready 상태인node 수를 카운트해서 저장
    # 이 문제는 일반 애플리케이션을 실행할수 있는 Node를 카운트 하라는 문제입니다.
    # normal workload의 의미 Pod를 이용해 일반적인 애플리케이션을 동작(특정 toleration들이 없는 파드들이 실행되는..)하는 node중 ready 상태인 노드 수 카운트
    
    # 1. 먼전 해당 hk8s 클러스터에 접속한후 ready 상태의 노드를 확인합니다.
    $ kubectl use-context hk8s
    $ kubectl get nodes | grep -i -w ready
    hk8s-master    Ready    control-plane   14h   v1.28.0
    hk8s-woker1    Ready    <none>          14h   v1.28.0
    hk8s-woker2    Ready    <none>          14h   v1.28.0
    
    # 2. 각 노드의 taint를 확인해서 pod tolerations를 요구하는 항목의 설정이 있는지 확인합니다.
    # master 시스템은 NoSchedule을 통해 normal workload 실행이 제한됩니다. 마스터에서는 일반 애플리케이션 파드가 스케쥴링 되지 않는다는 뜻입니다.
    $ kubectl describe nodes hk8s-master | grep -i taint
    Taints:             node-role.kubernetes.io/control-plane:NoSchedule
    
    # worker1번과 worker2번 노드에는 Taint가 설정되어 있지 않습니다. 그러니 normal workload가 실행가능합니다. 결국 이 두개가 답이 됩니다.
    $ kubectl describe nodes hk8s-woker1 | grep -i taint
    Taints:             <none>
    
    $ kubectl describe nodes hk8s-woker1 | grep -i taint
    Taints:             <none>
    
    # 3. echo 명령을 이용해서 2를 /var/CKA2022/count.txt 저장합니다.
    echo "2" > /var/CKA2022/count.txt
    ```


### **❓4. Management Node**

- Cluster: kubectl config use-context **k8s**

---

**Set the node named k8s-worker1 as unavailable and reschedule all the pods running on it.**

---

- ❗Answer

    ```bash
    kubectl drain k8s-worker1 --ignore-daemonsets  --force
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
- **답안**

    ```jsx
    # console에서 k8s master의 etcd 백업해서 console의 디렉토리에 snapshot 파일로 저장
    **# snapshot save**
    sudo ETCDCTL_API=3 etcdctl --endpoints=https://MasterIP:2379 \
        --cacert=/data/cka/ca.crt \
        --cert=/data/cka/server.crt \
        --key=/data/cka/server.key \
        snapshot save /data/etcd-snapshot.db
    
    **# RESTORE**
    # console에 etcd-snapshot-previous.db 파일이 존재하는 경우
    scp /data/etcd-snapshot-previous.db cluster_master'sIP:~ 
    ssh cluster_master'sIP
    **sudo ETCDCTL_API=3  etcdctl --data-dir=/var/lib/etcd-restore snapshot restore** etcd-snapshot-previous.db
    ls /var/lib/etcd-restore
    
    ## etcd yaml파일 수정
    sudo vi /etc/kubernetes/manifests/etcd.yaml
    …
     - hostPath:
          path: /var/lib/etcd-restore
          type: DirectoryOrCreate
        name: etcd-data
    
    # docker 명령으로 etcd가 restart 되었는지 확인
    #sudo docker ps -a | grep etcd
    # ctr -n k8s.io containers list | grep etcd
    crictl ps
    sudo crictl  logs etcd'sID
    exit
    
    ## console에서 쿠버네티스 동작 상태 확인
    kubectl get nodes
    ```


---

- 참고: 영상 학습과 동일한 예전 유형의 문제 풀이 - 작업 클러스터(**k8s)**
    
  ---

  First, create a snapshot of the existing etcd instance running at `https://127.0.0.1:2379`, saving the snapshot to `/data/etcd-snapshot.db`.
  Next, restore an existing, previous snapshot located at `/data/etcd-snapshot-previous.db`.

  The following TLS certificates/key are supplied for connecting to the server with etcdctl:
  CA certificate: `/etc/kubernetes/pki/etcd/ca.crt`
  Client certificate: `/etc/kubernetes/pki/etcd/server.crt`
  Client key: `/etc/kubernetes/pki/etcd/server.key`
    
  ---

    - ❗Answer

        ```bash
        ssh master
        sudo -i
        sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
           --cacert=/etc/kubernetes/pki/etcd/ca.crt \
           --cert=/etc/kubernetes/pki/etcd/server.crt \
           --key=/etc/kubernetes/pki/etcd/server.key \
           snapshot save /data/etcd-snapshot.db
        
        sudo ETCDCTL_API=3  etcdctl --data-dir=/var/lib/etcd-previous snapshot restore /data/etcd-snapshot-previous.db
        sudo tree /var/lib/etcd-previous/
        
        sudo vi /etc/kubernetes/manifests/etcd.yaml
        ...
          - hostPath:
              path: /var/lib/etcd-previous
              type: DirectoryOrCreate
            name: etcd-data
        
        # cri-docker 로 runtime 운영시 
        sudo docker ps -a | grep etcd; exit
        
        # containerd 로 실행시
        sudo ctr -n k8s.io container ls | grep etcd
        
        exit
        exit
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

- 답 : 1.28.0 → 1.28.2로 Ubuntu 기반 k8s  upgrade

  ### Control-plane의 k8s(kubeadm,kubelet,kubectl) 업그레이드

    - Control-plane  upgrade
    - **k8s cluster의 Control-plane을 버전 1.28.0 서 1.28.2 로 업그레이드하시오.**
    - [https://kubernetes.io/ko/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#업그레이드할-버전-결정](https://kubernetes.io/ko/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%93%9C%ED%95%A0-%EB%B2%84%EC%A0%84-%EA%B2%B0%EC%A0%95)

    ```bash
    ## k8s upgrade
    1. 업그레이드 할 시스템 접속
    **ssh k8s-master
    sudo -i
    kubectl get nodes**
    NAME          STATUS   ROLES           AGE   VERSION
    k8s-master    Ready    control-plane   20d   v1.28.0
    k8s-worker1   Ready    <none>          20d   v1.28.0
    k8s-worker2   Ready    <none>          20d   v1.28.0
    
    2. 업그레이드할 버전 결정및 확인. 1.28.8-00 버전을 검색
    **apt update
    apt-cache madison kubeadm** 
    
    # 상위 몇개 라인만 확인
    **apt-cache madison kubeadm | head**
       kubeadm |  1.28.2-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
       kubeadm |  1.28.1-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
       kubeadm |  1.28.0-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
    	...
    
    [컨트롤 플레인 노드 업그레이드]
    https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes
    # 1.28.0-00에서 1.28.2-00를 최신 패치 버전으로 바꾼다.
    1. kubeadm 업그레이드 :  kubeadm-1.28.2-00
    **apt-mark unhold kubeadm
    apt-get update
    apt-get install -y kubeadm=1.28.2-00
    apt-mark hold kubeadm
    
    kubeadm version**
    **kubeadm version: &version.Info{Major:"1", Minor:"28", GitVersion:"v1.28.2", GitCommit:"89a4ea3e1e4ddd7f7572286090359983e0387b2f", GitTreeState:"clean", BuildDate:"2023-09-13T09:34:32Z", GoVersion:"go1.20.8", Compiler:"gc", Platform:"linux/amd64"}**
    
    4. master componants 를 업그레이드
    **kubeadm upgrade plan**
    ...
    Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
    COMPONENT   CURRENT       TARGET
    kubelet     3 x v1.28.0   v1.28.8
    
    Upgrade to the latest version in the v1.28 series:
    
    COMPONENT                 CURRENT   TARGET
    kube-apiserver            v1.28.7   v1.28.8
    kube-controller-manager   v1.28.7   v1.28.8
    kube-scheduler            v1.28.7   v1.28.8
    kube-proxy                v1.28.7   v1.28.8
    CoreDNS                   v1.10.1   v1.10.1
    etcd                      3.5.9-0   3.5.9-0
    
    You can now apply the upgrade by executing the following command:
    
    	kubeadm upgrade apply v1.28.8
    
    Note: Before you can perform this upgrade, you have to update kubeadm to v1.28.8._____________________________________________________________________
    
    _____________________________________________________________________
    
    # update 가능한 마스터 컴포너트 확인하고 upgrade 실행
    **kubeadm upgrade apply v1.28.2 -y**
    [upgrade/config] Making sure the configuration is correct:
    [upgrade/config] Reading configuration from the cluster...
    [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [preflight] Running pre-flight checks.
    [upgrade] Running cluster health checks
    [upgrade/version] You have chosen to change the cluster version to "v1.28.2"
    [upgrade/versions] Cluster version: v1.28.7
    [upgrade/versions] kubeadm version: v1.28.2
    [upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
    [upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
    [upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
    W0316 16:40:55.105464   80167 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
    [upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.28.2" (timeout: 5m0s)...
    [upgrade/etcd] Upgrading to TLS for etcd
    [upgrade/staticpods] Preparing for "etcd" upgrade
    [upgrade/staticpods] Current and new manifests of etcd are equal, skipping upgrade
    [upgrade/etcd] Waiting for etcd to become available
    [upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests522589174"
    [upgrade/staticpods] Preparing for "kube-apiserver" upgrade
    [upgrade/staticpods] Renewing apiserver certificate
    [upgrade/staticpods] Renewing apiserver-kubelet-client certificate
    [upgrade/staticpods] Renewing front-proxy-client certificate
    [upgrade/staticpods] Renewing apiserver-etcd-client certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-03-16-16-40-55/kube-apiserver.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=kube-apiserver
    [upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
    [upgrade/staticpods] Renewing controller-manager.conf certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-03-16-16-40-55/kube-controller-manager.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=kube-controller-manager
    [upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
    [upgrade/staticpods] Preparing for "kube-scheduler" upgrade
    [upgrade/staticpods] Renewing scheduler.conf certificate
    [upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-03-16-16-40-55/kube-scheduler.yaml"
    [upgrade/staticpods] Waiting for the kubelet to restart the component
    [upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
    [apiclient] Found 1 Pods for label selector component=kube-scheduler
    [upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
    [upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config1418164023/config.yaml
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
    [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy
    
    [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.28.2". Enjoy!
    
    [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
    
    5. 노드 드레인 : console이나 control-plane(master)에서 실행
    **kubectl drain k8s-master --ignore-daemonsets** 
    # 삭제시 coredns delete pending 발생시 다른 터미널에서 
    # kubectl delete deployment coredns -n kube-system
    # kubectl delete pod --force -n kube-system coredns-xxx-XXX
    # 명령으로 수동 삭제 지원 필요 
    
    **kubectl get nodes**
    NAME          STATUS                     ROLES           AGE   VERSION
    k8s-master    Ready,SchedulingDisabled   control-plane   20d   v1.28.0
    k8s-worker1   Ready                      <none>          20d   v1.28.0
    k8s-worker2   Ready                      <none>          20d   v1.28.0
    
    6. kubelet과 kubectl 업그레이드
    **apt-mark unhold kubelet kubectl 
    apt-get update
    apt-get install -y kubelet=1.28.2-00 kubectl=1.28.2-00**
    **apt-mark hold kubelet kubectl**
    
    **systemctl daemon-reload
    systemctl restart kubelet**
    
    7. 노드 uncordon
    **kubectl uncordon k8s-master**
    
    kubectl get nodes
    NAME          STATUS   ROLES           AGE   VERSION
    **k8s-master    Ready    control-plane   20d   v1.28.2**
    k8s-worker1   Ready    <none>          20d   v1.28.0
    k8s-worker2   Ready    <none>          20d   v1.28.0
    
    **exit
    exit**
    ```


---

- ❗OLD Version : Answer

    ```bash
    # kubeadm 업그레이드
    sudo yum install -y kubeadm-1.25.3-0 --disableexcludes=kubernetes
    kubeadm version
    
    # node components 업그레이드
    sudo kubeadm upgrade plan v1.25.3
    sudo kubeadm upgrade apply v1.25.3
    
    # 노드 드레인
    kubectl drain hk8s-m --ignore-daemonsets 
    
    # kubelet과 kubectl 업그레이드
    sudo yum install -y kubelet-1.25.3-0 kubectl-1.25.3-0 --disableexcludes=kubernetes
    sudo systemctl daemon-reload
    sudo systemctl restart kubelet
    
    # 노드 uncordon
    sudo kubectl uncordon hk8s-m
    ```


### **❓7.** Authentication and Authorization

- Cluster : **k8s**

---

Context You have been asked to create a new ClusterRole for a deployment pipeline and bind it to a specific ServiceAccount scoped to a specific namespace.
**Task:**
- Create a new ClusterRole named `deployment-clusterrole`, which only allows to **create** the following resource types: **Deployment StatefulSet DaemonSet**
- Create a new ServiceAccount named `cicd-token` in the existing namespace `app-team1`.
- Bind the new ClusterRole `deployment-clusterrole` to the new ServiceAccount `cicd-token`, limited to the namespace `app-team1`.

---

- ❗Answer

    ```bash
    kubectl config use-context k8s
    
    kubectl create **clusterrole** deployment-clusterrole --verb=create --resource=deployment,statefulset,daemonset
    kubectl get clusterrole deployment-clusterrole
    
    kubectl create **serviceaccount** cicd-token --namespace=app-team1
    kubectl get serviceaccounts --namespace app-team1
    
    kubectl create **clusterrolebinding** **deployment-clusterrolebinding** --clusterrole=deployment-clusterrole --serviceaccount=app-team1:cide-token --namespace=app-team1 
    kubectl describe clusterrolebindings deployment-clusterrolebinding
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

---

- ❗Answer

    ```bash
    kubectl create namespace cka-exam
    kubectl get namespaces cka-exam 
    
    kubectl run pod-01 --image=busybox  --dry-run=client -o yaml > pod-01.yaml 
    vi pod-01.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-01
      namespace: cka-exam
    spec:
      containers:
      - env:
        - name: CERT
          value: CKA-cert
        image: busybox
        name: pod-01
        command: ["/bin/sh"]
        args: ["-c", "while true; do echo $(CERT); sleep 10;done"]
    
    kubectl apply -f pod-01.yaml
    kubectl get pod -n cka-exam
    ```


### **❓9.** multi-container Pod 생성

- cluster :  kubectl config use-context **hk8s**
- Create a pod with 4 containers running : nginx, redis and memcached
    - pod name: eshop-frontend
    - image: nginx
    - image: redis
    - image: memcached

- ❗Answer

    ```bash
    kubectl run eshop-frontend --image=nginx  --dry-run=client -o yaml > multi.yaml
    
    vi multi.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: eshop-frontend
    spec:
      containers:
      - image: nginx
        name: nginx
      - image: redis
        name: redis
      - image: memcached
        name: memcached
    
    kubectl apply -f multi.yaml
    ```


### **❓10.** Side-car Container Pod 실행

- 작업 클러스터 :  kubectl config use-context **k8s**
- An existing Pod needs to be integrated into the `Kubernetes built-in logging architecture` (e.g. `kubectl logs`).
- Adding a streaming sidecar container is a good and common way to accomplish this requirement.
- Task:
    - Add a sidecar container named `sidecar`, using the busybox Image, to the existing Pod `eshop-cart-app`.
    - The new `sidecar` container has to run the following `command: /bin/sh -c "tail -n+1 -f /var/log/cart-app.log"`
    - Use a Volume, mounted at `/var/log`, to make the log file `cart-app.log` available to the `sidecar` container.
    - Don't modify the `cart-app`.
- ❗Answer

    ```bash
    kubectl get pod eshop-cart-app -o yaml > sidecar.yaml
    cat sidecar.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      name: eshop-cart-app
    spec:
      containers:
      - image: busybox
        name: cart-app
        command: ['/bin/sh', '-c', 'i=1;while :;do  echo -e "$i: Price: $((RANDOM % 10000 + 1))" >> /var/log**/cart-app.log**; i=$((i+1)); sleep 2; done']
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - emptyDir: {}
        name: varlog
    
    **vi sidecar.yaml**
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
      **- name: price
        image: busybox
        command: [/bin/sh, -c, "tail -n+1 -f /var/log/cart-app.log"]
        volumeMounts:
        - name: varlog
          mountPath: /var/log**
      volumes:
      - emptyDir: {}
        name: varlog
    
    **kubectl delete pod eshop-cart-app
    kubectl apply -f sidecar.yaml**
    ```


### ❓11. Pod Scale-out

- Cluster: kubectl config use-contex **k8s**

---

Expand the number of running Pods in "**eshop-order**" to 5.

- namespace: devops
- deployment: eshop-order
- replicas: **5**

---

- ❗Answer

    ```bash
    kubectl get deployment -n devops
    kubectl get deployment eshop-order -n devops
    kubectl **scale** deployment eshop-order --replicas=5 --namespace devops
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

---

- ❗Answer

    ```bash
    # create
    kubectl create deployment  nginx-app --image=nginx:1.11.10-alpine --replicas=3 --dry-run=client -o yaml > deplyment.yaml
    cat deplyment.yaml 
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        app: nginx-app
      name: nginx-app
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: nginx-app
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: nginx-app
        spec:
          containers:
          - image: nginx:1.11.10-alpine
            name: nginx
            resources: {}
    status: {}
    
    kubectl apply -f deplyment.yaml --record
    
    **# rolling update**
    kubectl set image deployment nginx-app **nginx=nginx:1.11.13-alpine** **--record  # --record 옵션은 생략 가능합니다. docs를 보면서 진행하세요.**
    kubectl rollout history deployment **nginx-app
    
    # roll-back**
    kubectl rollout undo deployment nginx-app
    kubectl rollout history deployment **nginx-app**
    ```


### **❓13.** Network Policy with Namespace

- 작업 클러스터 : kubectl config use-context **k8s**

---

**Creat**e a new **NetworkPolicy** named allow-port-from-namespace in the existing **namespace devops**.

Ensure that the new NetworkPolicy allows Pods in namespace migops(using label **team=migops**) to connect to port 80 of Pods in namespace devops.

Further ensure that the new NetworkPolicy: does not allow access to Pods, which don't listen on port 80 does not allow access from Pods, which are not in namespace migops

---

- ❗Answer

    ```bash
    **kubectl get deployment -n devops**
    devops            Active   45d   kubernetes.io/metadata.name=devops,**team=devops**
    migops            Active   45d   kubernetes.io/metadata.name=migops,**team=migops**
    presales          Active   45d   kubernetes.io/metadata.name=presales,team=presales
    
    **vi policy.yaml**
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: allow-port-from-namespace
      **namespace: devops**
    spec:
      **podSelector: {}**   
      policyTypes:
      - Ingress
      ingress:
      - from:
        - namespaceSelector:
            matchLabels:
              team: migops
        ports:
        - protocol: TCP
          port:80
    
    **kubectl apply -f policy.yaml**
    ```


### **❓14. Create a persistent volume**

- Cluster: kubectl config use-context **k8s**

---

**Create a persistent volume** with name app-config, of capacity 1Gi and access mode ReadWriteMany.
The type of volume is hostPath and its location is /var/app-config.

---

- ❗Answer

    ```bash
    vi pv.yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: app-config
    spec:
      capacity:
        storage: 1Gi
      accessModes:
      - ReadWriteMany
      hostPath:
        path: /var/app-config
    
    kubectl apply -f pv.yaml
    kubectl get pv
    ```


### **❓15. Deploy and Service**

- 작업 클러스터 : kubectl config use-context **k8s**

---

Reconfigure the **existing** deployment `front-end` and add a **port specification named** `http` exposing port `80/tcp` of the existing container nginx.
Create a new service named `front-end-svc` exposing the container port http.
Configure the new service to also expose the individual Pods via a `NodePort` on the nodes on which they are scheduled

---

- ❗**Answer**

    ```bash
    kubectl get deploy front-end -o yaml > **front-end.yaml**
    
    cat > front-end.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: front-end
    spec:
      selector:
        matchLabels:
          app: front-end
      replicas: 2
      template:
        metadata:
          labels:
            app: front-end
        spec:
          containers:
          - name: http
            image: nginx
            ports:
            - name: http 
              containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: front-end-svc
    spec:
      type: NodePort
      ports:
      - port: 80
        protocol: TCP
        targetPort: http
      selector:
        app: front-end
    
    kubectl delete deploy front-end
    kubectl apply -f front-end.yaml
    kubectl get deployments.apps,svc
    ```


### **❓16.** DNS Lookup

- 작업 클러스터 : kubectl config use-context **k8s**

---

**Create a nginx pod** called nginx-resolver using image nginx, expose it internally with a service called nginx-resolver-service.
Test that you are able to look up the service and pod names from within the cluster. Use the image: busybox:1.28 for dns lookup.
- Record results in /var/CKA2022/nginx.svc and  /var/CKA2022/nginx.pod
- Pod: nginx-resolver created
- Service DNS Resolution recorded correctly
- Pod DNS resolution recorded correctly

---

- ❗**Answer**

    ```bash
    #**Create a nginx pod**
    **kubectl run nginx-resolver --image=nginx --port=80
    kubectl expose pod nginx-resolver --name nginx-resolver-service --port=80 
    kubectl get svc nginx-resolver-service** 
    	nginx-resolver-service   10.104.150.11
    # Test:Service DNS Resolution recorded correctly -> /var/CKA2022/nginx.svc
    **kubectl run test-nslookup --image=busybox:1.28 -it --restart=Never --rm -- nslookup  10.104.150.11 
    kubectl run test-nslookup --image=busybox:1.28 -it --restart=Never --rm -- nslookup  nginx-resolver-service > /var/CKA2022/nginx.svc**
    
    # Test: Pod DNS resolution recorded correctly -> /var/CKA2022/nginx.pod
    kubectl get pod ngnix-resolver -o wide
          nginx-resolver   **10.244.1.55**
    kubectl run test-nslookup --image=busybox:1.28 -it --restart=Never --rm -- nslookup  **10-244-1-55.default.pod.cluster.local** > **/var/CKA2022/nginx.pod**
    
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

---

- ❗Answer

    ```bash
    vi pvc.yaml
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
    
    kubectl apply -f pvc.yaml
    kubectl get pv,pvc
    
    vi pod-with-pvc.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: web-server
    spec:
      containers:
        - name: web-server
          image: nginx
          volumeMounts:
          - mountPath: "/usr/share/nginx/html"
            name: pv-volume
      volumes:
        - name: pv-volume
          persistentVolumeClaim:
            claimName: pv-volume
    
    kubectl **apply -f pod-with-pvc**
    kubectl get pod
    ```