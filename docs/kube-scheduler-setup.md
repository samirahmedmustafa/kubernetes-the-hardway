1. Generate kube-scheduler private key, certificate request and sign it with kubernetes CA (execute the below inside the CA directory)

```
    openssl genrsa -out kube-scheduler.key 2048
    openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler" -out kube-scheduler.csr
    openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-scheduler.crt -days 1000
```
 
```
    openssl verify -CAfile ca.crt kube-scheduler.crt
```
2. Distribute the certificates and binaries to the master servers

```
    scp kube-scheduler.crt kube-scheduler.key root@master-1:/var/lib/kubernetes/
    scp kube-scheduler.crt kube-scheduler.key root@master-2:/var/lib/kubernetes/
```

3. Create kubeconfig configuration file and distribute it to master servers

```
{
  ./kubectl config set-cluster home-cluster \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  ./kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  ./kubectl config set-context default \
    --cluster=home-cluster \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  ./kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

```
    scp kube-scheduler.kubeconfig root@master-1:/var/lib/kubernetes/
    scp kube-scheduler.kubeconfig root@master-2:/var/lib/kubernetes/
```

4. Create systemd service file and distribute to master servers

```
cat <<EOF | sudo tee kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --bind-address=127.0.0.1 \\
  --leader-elect=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
    scp kube-scheduler.service master-1:/etc/systemd/system/
    scp kube-scheduler.service master-2:/etc/systemd/system/
```

5. Download kube-apiserver binary and distribute to control plane master servers

```
    wget https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-scheduler
    chmod +x kube-scheduler
    scp kube-scheduler root@master-1:/usr/local/bin/
    scp kube-scheduler root@master-2:/usr/local/bin/
```

6. Restart kube-scheduler service

```
{
  ssh master-1 sudo systemctl daemon-reload
  ssh master-1 sudo systemctl enable --now kube-scheduler
  ssh master-2 sudo systemctl daemon-reload
  ssh master-2 sudo systemctl enable --now kube-scheduler
}
```
[Previous: Setup kube-apiserver](kube-apiserver-setup.md)&nbsp;&nbsp;&nbsp;[Next: Setup kube-controller-manager](kube-controller-manager-setup.md)
