1. Create the kube-proxy configuration file

```
cat > /var/lib/kubernetes/kube-proxy-config.yaml <<EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kubeconfig"
mode: "iptables"
clusterCIDR: "192.168.1.0/24"
EOF
```

2. Create the service file
```
cat > /etc/systemd/system/kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kubernetes/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
    scp /etc/systemd/system/kube-proxy.service worker-2:/etc/systemd/system/
```
3. 
