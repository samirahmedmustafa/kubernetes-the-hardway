#Create 2 certificate authorities the first is for etcd and the second is for controller plane cluster

#0. Install openssl, tar and wget

```
    dnf install -y openssl wget tar
```

#1. etcd CA generation

```
    openssl genrsa -out etcd-ca.key 2048
    openssl req -new -key etcd-ca.key -subj "/CN=ETCD-CA" -out etcd-ca.csr
    openssl x509 -req -in etcd-ca.csr -signkey etcd-ca.key -CAcreateserial  -out etcd-ca.crt -days 1000
```

#2. kubernetes CA generation

```
    openssl genrsa -out ca.key 2048
    openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
    openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 1000
```
