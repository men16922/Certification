# 9. CNI (Container Network Interface) 2025

Weightage: 10%

Install and set up a Container Network Interface (CNI) that meets these requirements:
Pick and install one of the CNI options:

The CNI you choose must satisfy following requirement:
<native network policy support>

Flannel version 0.26.1

Manifest - https://github.com/flanner-io/flanner/releases/download/v0.26.1/kube-flanner.yml

Calico version 3.28.2

Manifest - https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml

```