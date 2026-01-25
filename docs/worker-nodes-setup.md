1. Disable swap

```
       ssh worker-1 sudo swapoff -a
       ssh worker-1 sudo sed -ie 's/.*swap.*defaults/#&/' /etc/fstab
       ssh worker-1 sudo swapoff -a
       ssh worker-2 sudo sed -ie "s/.*swap.*defaults/#&/" /etc/fstab
```
2. Download kubernetes node binaries, containerd and CNI
```
    wget -q --show-progress --https-only --timestamping \
        https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubelet \
        https://github.com/containerd/containerd/releases/download/v2.2.1/containerd-2.2.1-linux-amd64.tar.gz \
        https://github.com/containernetworking/plugins/releases/download/v1.9.0/cni-plugins-linux-amd64-v1.9.0.tgz \
        https://github.com/opencontainers/runc/releases/download/v1.4.0/runc.amd64      
```

```
    mkdir containerd/ cni/
    tar -xzf containerd-2.2.1-linux-amd64.tar.gz -C containerd
    tar -xzf cni-plugins-linux-amd64-v1.9.0.tgz -C cni/
    scp cni/* containerd/* root@worker-1:/usr/local/bin/
    scp runc.amd64 root@worker-1:/usr/local/sbin/runc
    scp cni/* containerd/* root@worker-2:/usr/local/bin/
    scp runc.amd64 root@worker-2:/usr/local/sbin/runc
```

3. Change binaries to execution in the 2 worker nodes
```
    chmod +x kubelet
    scp kubelet root@worker-1:/usr/local/bin/
    scp kubelet root@worker-2:/usr/local/bin/
```

4. Create kubelet config.yaml
```
cat > config.yaml <<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.crt"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.96.0.10"
resolvConf: "/etc/resolv.conf"
runtimeRequestTimeout: "15m"
EOF
```

```
    scp config.yaml root@worker-1:/var/lib/kubernetes/
    scp config.yaml root@worker-2:/var/lib/kubernetes/
```

5. Create kubelet systemd service file
```
cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --kubeconfig=/var/lib/kubernetes/kubeconfig \\
  --bootstrap-kubeconfig=/var/lib/kubernetes/bootstrap-kubeconfig \\
  --config=/var/lib/kubernetes/config.yaml \\
  --rotate-certificates=true \\
  --rotate-server-certificates=true \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
    scp kubelet.service root@worker-1:/etc/systemd/system/
    scp kubelet.service root@worker-2:/etc/systemd/system/
```

6. Generate the token kubeconfig file for workers

```
    token=`ssh master-1 "sudo /usr/local/bin/kubectl -n kube-system get secrets -o jsonpath='{.items[?(@.type==\"bootstrap.kubernetes.io/token\")].metadata.name}' --kubeconfig admin.kubeconfig"`
    token_id=$(echo ${token} | awk -F'-' '{ print $NF}')
    secret64=`ssh master-1 "/usr/local/bin/kubectl -n kube-system get secret bootstrap-token-${token_id} -o jsonpath='{.data.token-secret}' --kubeconfig admin.kubeconfig"`
    decoded_secret=$(echo ${secret64} | base64 -d)
    tkn=$(echo ${token_id}.${decoded_secret})
```
```
cat > bootstrap-kubeconfig <<EOF
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /var/lib/kubernetes/ca.crt
    server: https://192.168.1.50:6443
  name: bootstrap
contexts:
- context:
    cluster: bootstrap
    user: kubelet-bootstrap
  name: bootstrap
current-context: bootstrap
preferences: {}
users:
- name: kubelet-bootstrap
  user:
    token: ${tkn}
EOF
```
7. Deploy containerd systemd service file
```
cat > containerd.service <<EOF
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target dbus.service

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF
```

```
    scp containerd.service root@worker-1:/etc/systemd/system/
    scp containerd.service root@worker-2:/etc/systemd/system/
```

```
    ssh worker-1 sudo systemctl daemon-reload
    ssh worker-1 sudo systemctl enable --now containerd kubelet
    ssh worker-2 sudo systemctl daemon-reload
    ssh worker-2 sudo systemctl enable --now containerd kubelet
```
[Setup kube-controller-manager](kube-controller-manager-setup.md)&nbsp;&nbsp;&nbsp;&nbsp;[Next: Setup (cilium) networking](networking-setup.md)
