## RBAC for Kubelet Authorization

In this section you will configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving node's status, metrics, logs (i.e. `kubectl logs <pod>`), and executing commands in pods (`kubectl exec -it <pod>`).

> This tutorial sets the Kubelet `--authorization-mode` flag to `Webhook`. Webhook mode uses the [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API to determine authorization.


Create the `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) with permissions to access the Kubelet API and perform most common tasks associated with managing pods:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```
Reference: https://v1-12.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole

The Kubernetes API Server authenticates to the Kubelet as the `kube-apiserver` user using the client certificate as defined by the `--kubelet-client-certificate` flag.

```
*kube-apiserver (client) -> kubelet (server)
--kubelet-client-certificate=/var/lib/kubernetes/kube-apiserver.crt
--kubelet-client-key=/var/lib/kubernetes/kube-apiserver.key

*Check the CN/username
master-1 $ openssl x509 -in /var/lib/kubernetes/kube-apiserver.crt -text
```

Bind the `system:kube-apiserver-to-kubelet` ClusterRole to the `kube-apiserver` user:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kube-apiserver
EOF
```
Reference: https://v1-12.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding

Next: [DNS Addon](14-dns-addon.md)
