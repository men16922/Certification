# 12. 모든 Custom CRD 나열 및 custom-crd.txt 저장
Weightage: 7%

List all custom crd from cert manager and store it in custom-crd.txt.

Get the subject field from the cert manager and store it in cert-manager-subject.txt

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.crds.yaml
kubectl get crd | grep cert > custom-crd.txt

kubectl get crd certificates.cert-manager.io -o jsonpath={.spec.versions[*].schema.openAPIV3Schema.properties.spec.properties.subject} \
> cert-manager-subject.txt
cat cert-manager-subject.txt

```