# Dynamic Kubelet Configuration (i.e. dynamically update changes to kubelet configuration file without manually restarting the kubelet service)

***______
Ref: https://insujang.github.io/2020-08-24/dynamic-kubelet-configuration/
Kubelet, at launch time, loads configuration files from pre-specified files. Changed configurations are not applied into the running Kubelet process during runtime, hence manual restarting Kubelet is required after modification.
Dynamic Kubelet configuration eliminates this burden, making Kubelet monitors its configuration changes and restarts when it is updated. It uses Kubernetes a ConfigMap object.
***_____

`sudo apt install -y jq`


```
NODE_NAME="worker-1"; NODE_NAME="worker-1"; curl -sSL "https://localhost:6443/api/v1/nodes/${NODE_NAME}/proxy/configz" -k --cert admin.crt --key admin.key | jq '.kubeletconfig|.kind="KubeletConfiguration"|.apiVersion="kubelet.config.k8s.io/v1beta1"' > kubelet_configz_${NODE_NAME}
```

```
kubectl -n kube-system create configmap nodes-config --from-file=kubelet=kubelet_configz_${NODE_NAME} --append-hash -o yaml
```

Edit node to use the dynamically created configuration
```
kubectl edit worker-2
```

Configure Kubelet Service

Create the `kubelet.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --bootstrap-kubeconfig="/var/lib/kubelet/bootstrap-kubeconfig" \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --dynamic-config-dir=/var/lib/kubelet/dynamic-config \\
  --cert-dir= /var/lib/kubelet/ \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
