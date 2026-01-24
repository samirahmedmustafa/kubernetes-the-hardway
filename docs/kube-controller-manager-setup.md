1. Generate kube-controller-manager private key, certificate request and sign it with kubernetes CA

```
    openssl genrsa -out kube-controller-manager.key 2048
    openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager" -out kube-controller-manager.csr
    openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-controller-manager.crt -days 1000
``` 

```
    openssl verify -CAfile ca.crt kube-controller-manager.crt
```

2. Distribute the certificates and binaries to the master servers

```
    scp kube-controller-manager.crt kube-controller-manager.key root@master-1:/var/lib/kubernetes/
    scp kube-controller-manager.crt kube-controller-manager.key root@master-2:/var/lib/kubernetes/
```

3. Create kubeconfig configuration file and distribute it to master servers

```
{
  ./kubectl config set-cluster home-cluster \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  ./kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  ./kubectl config set-context default \
    --cluster=home-cluster \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  ./kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

```
    scp kube-controller-manager.kubeconfig root@master-1:/var/lib/kubernetes/
    scp kube-controller-manager.kubeconfig root@master-2:/var/lib/kubernetes/
```

4. Create systemd service file

```
cat <<EOF | sudo tee kube-controller-manager.service
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
  --controllers=*,bootstrapsigner,tokencleaner \\
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
    scp kube-controller-manager.service root@master-1:/etc/systemd/system/
    scp kube-controller-manager.service root@master-2:/etc/systemd/system/
```

5. Download kube-apiserver binary and distribute to control plane master servers
```
    wget https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-controller-manager
    chmod +x kube-controller-manager
    scp kube-controller-manager root@master-1:/usr/local/bin/
    scp kube-controller-manager root@master-2:/usr/local/bin/
```

6. Start kube-controller-manager service
```
    for i in master-1 master-2; do
        ssh $i sudo systemctl daemon-reload
        ssh $i sudo systemctl enable --now kube-controller-manager
    done
```

7. Bootstrap Token Secret and kubelet
 7.a Bootstrap Token Secret Format
```
TOKEN_ID=$(openssl rand -hex 3)
TOKEN_SECRET=$(openssl rand -hex 8)
FULL_TOKEN="$TOKEN_ID.$TOKEN_SECRET"

cat > bootstrap-token.yml <<EOF
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
    token: ${FULL_TOKEN}
EOF
```

```
    scp bootstrap-kubeconfig worker-1:/var/lib/kubernetes/
    scp bootstrap-kubeconfig worker-2:/var/lib/kubernetes/
```

```
    ssh master-1 sudo /usr/local/bin/kubectl apply -f bootstrap-token.yml --kubeconfig admin.kubeconfig
```
 7.b Enable bootstrapping nodes to create CSR

```
cat > create_csrs_for_bootstrapping.yaml <<EOF

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
    scp create_csrs_for_bootstrapping.yaml master-1:
```
```
    ssh master-1 sudo /usr/local/bin/kubectl apply -f create_csrs_for_bootstrapping.yaml --kubeconfig admin.kubeconfig
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
    scp auto_approve_csrs.yaml master-1:
```
```
    ssh master-1 sudo /usr/local/bin/kubectl apply -f auto_approve_csrs.yaml --kubeconfig admin.kubeconfig
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
    scp auto_approve_renewals.yaml master-1:
```
```
    ssh master-1 sudo /usr/local/bin/kubectl apply -f auto_approve_renewals.yaml --kubeconfig admin.kubeconfig
```

[Previous: Setup kube-apiserver](kube-apiserver-setup.md)&nbsp;&nbsp;&nbsp;&nbsp;[Setup kubelet and kube-proxy in worker nodes](worker-nodes-setup.md)
