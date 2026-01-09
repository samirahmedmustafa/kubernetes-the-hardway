
1. Generate kube-apiserver private key, certificate request and sign it with kubernetes CA (execute the below inside the CA directory)

```
    openssl genrsa -out kube-apiserver.key 2048
    openssl req -new -key kube-apiserver.key -subj "/CN=kube-apiserver" -out kube-apiserver.csr -config openssl.cnf
    openssl x509 -req -in kube-apiserver.csr -CA /var/lib/kubernetes/ca.crt -CAkey /var/lib/kubernetes/ca.key -CAcreateserial  -out kube-apiserver.crt -extensions v3_req -extfile openssl.cnf -days 1000
```

```
    openssl verify -CAfile /var/lib/kubernetes/ca.crt kube-apiserver.crt
```
2. Create an encryption key to be used for encryption at rest
``` 
    ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF

```

3. Create a configuration file with kube-apiserver alternative names

```
cat > openssl.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 192.168.1.50
IP.3 = 192.168.1.51
IP.4 = 192.168.1.52
IP.5 = 127.0.0.1
EOF
```

4. Distribute the certificates to the master servers kubernetes directory

```
    cp kube-apiserver.key kube-apiserver.crt encryption-config.yaml /var/lib/kubernetes/
    scp kube-apiserver.key kube-apiserver.crt encryption-config.yaml master-2:/var/lib/kubernetes/
```

5. Create kube-apiserver service file (in each master server)

```
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=192.168.1.51 \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --enable-admission-plugins=NodeRestriction,ServiceAccount \\
  --enable-bootstrap-token-auth \\
  --etcd-cafile=/etc/etcd/etcd-ca.crt \\
  --etcd-certfile=/etc/etcd/etcd-server1.crt \\
  --etcd-keyfile=/etc/etcd/etcd-server1.key \\
  --etcd-servers=https://192.168.1.51:2379,https://192.168.1.52:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.crt \\
  --kubelet-client-certificate=/var/lib/kubernetes/kube-apiserver.crt \\
  --kubelet-client-key=/var/lib/kubernetes/kube-apiserver.key \\
  --runtime-config=api/all=true \\
  --service-account-issuer=https://192.168.1.51:6443 \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account.key \\
  --service-account-key-file=/var/lib/kubernetes/service-account.crt \\
  --service-cluster-ip-range=10.96.0.0/12 \\
  --service-node-port-range=30000-32767 \\
  --client-ca-file=/var/lib/kubernetes/ca.crt \\
  --tls-cert-file=/var/lib/kubernetes/kube-apiserver.crt \\
  --tls-private-key-file=/var/lib/kubernetes/kube-apiserver.key \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
    sed -e 's/192.168.1.51/192.168.1.52/' -e 's/etcd-server1/etcd-server2/' /etc/systemd/system/kube-apiserver.service > kube-apiserver.service
    scp kube-apiserver.service master-2:/etc/systemd/system/kube-apiserver.service
```
6. Download kube-apiserver binary and distribute to control plane master servers

```
    wget https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-apiserver
    chmod +x kube-apiserver
    mv kube-apiserver /usr/local/bin/
    scp /usr/local/bin/kube-apiserver master-2:/usr/local/bin/
```

7. Start kube-apiserver service

```
{
    systemctl daemon-reload
    systemctl enable --now kube-apiserver
    ssh master-2 systemctl daemon-reload
    ssh master-2 systemctl enable --now kube-apiserver
}
```
[Previous: Setup haproxy loadbalancer](lb-setup.md)&nbsp;&nbsp;&nbsp;&nbsp;[Next: Setup kube-scheduler](kube-scheduler-setup.md)
