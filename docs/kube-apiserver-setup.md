1. Create a configuration file with kube-apiserver alternative names

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
2. Generate kube-apiserver private key, certificate request and sign it with kubernetes CA (execute the below inside the CA directory)

```
    openssl genrsa -out kube-apiserver.key 2048
    openssl req -new -key kube-apiserver.key -subj "/CN=kube-apiserver" -out kube-apiserver.csr -config openssl.cnf
    openssl x509 -req -in kube-apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-apiserver.crt -extensions v3_req -extfile openssl.cnf -days 1000
```

```
    openssl verify -CAfile ca.crt kube-apiserver.crt
```
- Create front-proxy-ca to be used in the extra options needed in the extension-apiserver-authentication configmap, which would be needed later by the metrics server
  ```
    openssl genrsa -out front-proxy-ca.key 2048
    openssl req -x509 -new -nodes -key front-proxy-ca.key -subj "/CN=front-proxy-ca" -out front-proxy-ca.crt
  ``` 
3. Create an encryption key to be used for encryption at rest
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

4. Distribute the certificates to the master servers kubernetes directory

```
    scp front-proxy-ca.crt front-proxy-ca.key kube-apiserver.key kube-apiserver.crt encryption-config.yaml root@master-1:/var/lib/kubernetes/
    scp front-proxy-ca.crt front-proxy-ca.key kube-apiserver.key kube-apiserver.crt encryption-config.yaml root@master-2:/var/lib/kubernetes/
```

5. Create kube-apiserver service file (in each master server)

```
cat <<EOF | sudo tee kube-apiserver1.service
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
  --requestheader-allowed-names=front-proxy-client \\
  --requestheader-client-ca-file=/var/lib/kubernetes/front-proxy-ca.crt \\
  --requestheader-extra-headers-prefix=X-Remote-Extra- \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-username-headers=X-Remote-User \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
    sed -e 's/192.168.1.51/192.168.1.52/' -e 's/etcd-server1/etcd-server2/' kube-apiserver1.service > kube-apiserver2.service
```
```
    scp kube-apiserver1.service root@master-1:/etc/systemd/system/kube-apiserver.service
    scp kube-apiserver2.service root@master-2:/etc/systemd/system/kube-apiserver.service
```
6. Download kube-apiserver binary and distribute to control plane master servers

```
    wget https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-apiserver
    chmod +x kube-apiserver
    scp kube-apiserver root@master-1:/usr/local/bin/
    scp kube-apiserver root@master-2:/usr/local/bin/
```

7. Start kube-apiserver service

```
{
    ssh master-1 sudo systemctl daemon-reload
    ssh master-1 sudo systemctl enable --now kube-apiserver
    ssh master-2 sudo systemctl daemon-reload
    ssh master-2 sudo systemctl enable --now kube-apiserver
}
```

8. Create a cluster role binding for kube-apiserver to kubelet proxy
```
    ssh master-1 sudo /usr/local/bin/kubectl create clusterrolebinding kube-apiserver-to-kubelet --clusterrole=system:kubelet-api-admin --user=kube-apiserver --kubeconfig admin.kubeconfig
    ssh master-1 sudo /usr/local/bin/kubectl auth can-i get nodes/proxy --as kube-apiserver --kubeconfig admin.kubeconfig
```
[Previous: Setup haproxy loadbalancer](lb-setup.md)&nbsp;&nbsp;&nbsp;&nbsp;[Next: Setup kube-scheduler](kube-scheduler-setup.md)
