#Create 2 certificate authorities the first is for etcd and the second is for controller plane cluster

#0. Install openssl, tar and wget (preferrably in all servers)

- Install pkgs

```
    dnf install -y openssl wget tar
```

- (Optional) setup ssh key authenticate from master-1 to all other servers to transfer files/certs convenientely 
- (Optional) setup ssh key authenticate from master-1 to all other servers to transfer files/certs convenientely 
- (Optional) update /etc/hosts in all  servers as below

```
    cat /etc/hosts

        192.168.1.50 lb
        192.168.1.51 master-1
        192.168.1.52 master-2
        192.168.1.53 worker-1
        192.168.1.54 worker-2

```

- In master-1, create the below directories to organize the files

```
    mkdir kube-apiserver  kube-scheduler kube-controller-manager kubernetes-CA etcd
```

- Download kubectl

```
    wget https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubectl
    mv kubectl /usr/local/bin
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


=======
#3. Download kubectl and distribute to all server used for administration

```
    wget https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubectl
```

#These certificates needs to be kept as they will be used to sign future certificates, so basically they will be part of your local Certificate Authority

[next: kube-apiserver setup](kube-apiserver-setup.md)
