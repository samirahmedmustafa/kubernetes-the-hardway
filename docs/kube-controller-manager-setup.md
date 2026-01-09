1. Generate kube-controller-manager private key, certificate request and sign it with kubernetes CA

```
    openssl genrsa -out kube-controller-manager.key 2048
    openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager" -out kube-controller-manager.csr
    openssl x509 -req -in kube-controller-manager.csr -CA /var/lib/kubernetes/ca.crt -CAkey /var/lib/kubernetes/ca.key -CAcreateserial -out kube-controller-manager.crt -days 1000
``` 

```
    openssl verify -CAfile /var/lib/kubernetes/ca.crt kube-controller-manager.crt
```

2. Distribute the certificates and binaries to the master servers

```
    cp kube-controller-manager.crt kube-controller-manager.key /var/lib/kubernetes/
    scp kube-controller-manager.crt kube-controller-manager.key master-2:/var/lib/kubernetes/
```

3. Create kubeconfig configuration file and distribute it to master servers

```
{
  kubectl config set-cluster home-cluster \
    --certificate-authority=/var/lib/kubernetes/ca.crt \
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
  --bind-address=0.0.0.0 \\
  --cluster-cidr=192.168.1.0/24 \\
  --cluster-name=home-cluster \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.crt \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca.key \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.crt \\
  --controllers=*,tokencleaner \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.crt \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca.key \\
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
    ssh master-2 chmod +x /usr/local/bin/kube-controller-manager
```

6. Start kube-controller-manager service

```
    {
        systemctl daemon-reload
        systemctl enable --now kube-controller-manager 
        ssh master-2 systemctl daemon-reload
        ssh master-2 systemctl enable --now kube-controller-manager 
    }
```

7. Bootstrap Token Secret Format

```
TOKEN_ID=$(openssl rand -hex 3)
TOKEN_SECRET=$(openssl rand -hex 8)
FULL_TOKEN="$TOKEN_ID.$TOKEN_SECRET"
```
```
cat > bootstrap-token-${TOKEN_ID} <<EOF
apiVersion: v1
kind: Secret
metadata:
  # Name MUST be of form "bootstrap-token-<token id>"
  name: bootstrap-token-${TOKEN_ID}
  namespace: kube-system

# Type MUST be 'bootstrap.kubernetes.io/token'
type: bootstrap.kubernetes.io/token
stringData:

  # Token ID and secret. Required.
  token-id: ${TOKEN_ID}
  token-secret: ${TOKEN_SECRET}

  # Expiration. Optional.
  expiration: 2037-03-10T03:22:11Z

  # Allowed usages.
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"

  # Extra groups to authenticate the token as. Must start with "system:bootstrappers:"
  auth-extra-groups: system:bootstrappers:worker,system:bootstrappers:ingress
EOF

```

```
    kubectl apply -f bootstrap-token-${TOKEN_ID}
```

7. Enable bootstrapping nodes to create CSR

```
cat > bootstrapping_crb.yaml <<EOF

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: create-csrs-for-bootstrapping
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:node-bootstrapper
  apiGroup: rbac.authorization.k8s.io
EOF
```

```
    kubectl apply -f create_csrs_for_bootstrapping.yaml
```

8. Approve all CSRs for the group "system:bootstrappers"

```
cat > auto_approve_csrs.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-csrs-for-group
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
  apiGroup: rbac.authorization.k8s.io
EOF
```

```
    kubectl apply -f auto_approve_csrs.yaml
```

9. # Approve renewal CSRs for the group "system:nodes"

```
cat > auto_approve_renewals.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-renewals-for-nodes
subjects:
- kind: Group
  name: system:nodes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
  apiGroup: rbac.authorization.k8s.io
EOF
```

```
    kubectl apply -f auto_approve_renewals.yaml
```

[Previous: Setup kube-apiserver](kube-apiserver-setup.md)&nbsp;&nbsp;&nbsp;&nbsp;[Setup kubelet and kube-proxy in worker nodes](worker-nodes-setup.md)
