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

2. Generate kube-apiserver private key, certificate request and sign it with kubernetes CA

```
    openssl genrsa -out kube-apiserver.key 2048
    openssl req -new -key kube-apiserver.key -subj "/CN=kube-apiserver" -out kube-apiserver.csr -config openssl.cnf
    openssl x509 -req -in kube-apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-apiserver.crt -extensions v3_req -extfile openssl.cnf -days 1000
``` 

3. Create an encryption key to be used for encryption at rest

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

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
    mkdir -p /var/lib/kubernetes/
    cp kube-apiserver.key kube-apiserver.crt ca.crt /var/lib/kubernetes/
    ssh master-2 mkdir -p /var/lib/kubernetes/
    scp kube-apiserver.key kube-apiserver.crt ca.crt master-2:/var/lib/kubernetes/
```

5. Download kube-apiserver binary and distribute to control plane master servers

```
    wget https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-apiserver
    chmod +x kube-apiserver
    mv kube-apiserver /usr/local/bin/
    scp /usr/local/bin/kube-apiserver master-2:/usr/local/bin/
```


