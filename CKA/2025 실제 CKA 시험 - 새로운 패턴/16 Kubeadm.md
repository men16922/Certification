A kubeadm provisioned cluster was migrated to a new machine. Requires configuration changes to run successfully.

Task:
We need to fix a single-node cluster that got broken during machine migration.

Identify the broken cluster components and investigate what caused them to break.

The decommissioned cluster used an external etcd server.

Next, fix the configuration of all broken cluster components.

Ensure to restart all necessary services and components for changes to take effect.

Finally, ensure the cluster, single node and all pods are Ready.

```bash
kubectl get po
journalctl -u kubelet -f
ls /etc/kubernetes/manifests/

vi /etc/kubernetes/manifests/kube-apiserver

# --etcd-server=https://128.0.0.1:2379

sudo systemctl restart kubelet
sudo systemctl status kubelet

```