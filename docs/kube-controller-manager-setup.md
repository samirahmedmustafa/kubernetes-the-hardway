1. Generate kube-controller-manager private key, certificate request and sign it with kubernetes CA

```
    openssl genrsa -out kube-controller-manager.key 2048
    openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager" -out kube-controller-manager.csr
    openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-controller-manager.crt -days 1000
``` 

2. Distribute the certificates and binaries to the master servers

```
    cp kube-controller-manager.crt kube-controller-manager.key ca.crt /var/lib/kubernetes/
    scp kube-controller-manager.crt kube-controller-manager.key ca.crt master-2:/var/lib/kubernetes/
```

3. Create kubeconfig configuration file and distribute it to master servers

```
{
  kubectl config set-cluster home-cluster \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=home-cluster \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

```
    cp kube-controller-manager.kubeconfig /var/lib/kubernetes/
    scp kube-controller-manager.kubeconfig master-2:/var/lib/kubernetes/
```

4. Create systemd service file

```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=192.168.1.0/24 \\
  --cluster-name=home-cluster \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.crt \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca.key \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.crt \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account.key \\
  --service-cluster-ip-range=10.96.0.0/12 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
    scp /etc/systemd/system/kube-controller-manager.service master-2:/etc/systemd/system/
```

5. Download kube-apiserver binary and distribute to control plane master servers

```
    wget https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-controller-manager
    chmod +x kube-controller-manager
    mv kube-controller-manager /usr/local/bin/
    scp /usr/local/bin/kube-controller-manager master-2:/usr/local/bin/
```

6. Start kube-controller-manager service

```
    {
        systemctl daemon-reload
        systemctl enable --now kube-controller-manager 
    }
```

[Previous: Setup kube-apiserver](kube-apiserver-setup.md)&nbsp;&nbsp;[Setup kubelet and kube-proxy in worker nodes](worker-nodes-setup.md)
