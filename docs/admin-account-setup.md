# For the admin, the first account we will use the below procedure (execute the below inside the CA directory)

1. Create and sign samir certificates

```
    openssl genrsa -out admin.key 2048
    openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr
    openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out admin.crt -days 1000

```

2.

```
{
    kubectl config set-cluster home-cluster \
      --certificate-authority=ca.crt \
      --embed-certs=true \
      --server=https://192.168.1.50:6443 \
      --kubeconfig=admin.kubeconfig

    kubectl config set-credentials admin \
      --client-certificate=admin.crt \
      --client-key=admin.key \
      --embed-certs=true \
      --kubeconfig=admin.kubeconfig

    kubectl config set-context default \
      --cluster=home-cluster \
      --user=admin \
      --kubeconfig=admin.kubeconfig

    kubectl config use-context default --kubeconfig=admin.kubeconfig
}

```

# For the extra accounts (e.g. Samir), we will use the below process

1. Create samir private key and csr

1. Create and sign samir certificates

```
    openssl genrsa -out samir.key 2048
    openssl req -new -key samir.key -subj "/CN=samir/O=system:masters" -out samir.csr
```

2. Create a Kubernetes CertificateSigningRequest

```
    req=$(cat samir.csr | base64 | tr -d "\n")
```

```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
    name: samir
spec:
  # This is an encoded CSR. Change this to the base64-encoded contents of samir.csr
    request: ${req}
    signerName: kubernetes.io/kube-apiserver-client
    expirationSeconds: 8640000  
    usages:
        - client auth
EOF

```

3. Approve the CertificateSigningRequest

```
    kubectl get csr
    kubectl certificate approve samir
    kubectl get csr samir -o jsonpath='{.status.certificate}'| base64 -d > samir.crt
```

4. Configure samir certificate into kubeconfig

```
{
    kubectl config set-cluster home-cluster \
      --certificate-authority=ca.crt \
      --embed-certs=true \
      --server=https://192.168.1.50:6443 \
      --kubeconfig=samir.kubeconfig
    
    kubectl config set-credentials samir \
      --client-certificate=samir.crt \
      --client-key=samir.key \
      --embed-certs=true \
      --kubeconfig=samir.kubeconfig
    
    kubectl config set-context default \
      --cluster=home-cluster \
      --user=samir \
      --kubeconfig=samir.kubeconfig
    
    kubectl config use-context default --kubeconfig=samir.kubeconfig
}

```

5. Test

```
    kubectl --context samir auth whoami
```


