# 11. Kubernetes Context - 1

# Q19
Weightage: 7%

You have access to multiple clusters from your main terminal through kubectl contexts.
Write all those context names into
/opt/course/1/contexts

Next write a command to display the current context into
/opt/course/1/context_default_kubectl.sh,
the command should use kubectl.

Finally write a second command doing the same thing into
/opt/course/1/context_default_no_kubectl.sh,
but without the use of kubectl.

- [https://kubernetes.io/ko/docs/concepts/configuration/organize-cluster-access-kubeconfig/](https://kubernetes.io/ko/docs/concepts/configuration/organize-cluster-access-kubeconfig/)

```bash
mkdir -p /opt/course/1

kubectl config get-contexts -o name > /opt/course/1/contexts
cat /opt/course/1/contexts

echo kubectl config current-context > /opt/course/1/context_default_kubectl.sh
chmod +x /opt/course/1/context_default_kubectl.sh
cat /opt/course/1/context_default_kubectl.sh
bash /opt/course/1/context_default_kubectl.sh

cat ~/.kube/config | grep current
echo 'cat ~/.kube/config | grep current' > /opt/course/1/context_default_no_kubectl.sh
bash /opt/course/1/context_default_no_kubectl.sh
```