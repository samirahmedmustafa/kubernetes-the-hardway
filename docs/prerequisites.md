# From an additional server do the below activities

## Create 2 certificate authorities the first is for etcd and the second is for controller plane cluster

#0. Install openssl, tar and wget (preferrably in all servers)
   
- Set selinux to permissive

```
    setenforce 0
    grep ^SELINUX= /etc/selinux/config
        
        SELINUX=permissive
```

- Add firewall according to the below

```
    for i in 2380/tcp 2379/tcp 6443/tcp; do
        ssh master-1 sudo firewall-cmd --add-port=${i}
        ssh master-1 sudo firewall-cmd --permanent --add-port=${i}
        ssh master-2 sudo firewall-cmd --add-port=${i}
        ssh master-2 sudo firewall-cmd --permanent --add-port=${i}
    done
```

 <table>
    <tr><th>Service</th><th>PORT</th></tr>
    <tr><td>etcd</td><td>2380/tcp</td></tr>
    <tr><td>etcd</td><td>2379/tcp</td></tr>
    <tr><td>kube-apiserver</td><td>6443/tcp</td></tr>
    <tr><td>hubble-relay</td><td>10250/tcp</td></tr>
</table

- Install pkgs

```
    for i in master-1 master-2 worker-1 worker-2 lb; do
       ssh $i sudo dnf install -y -q openssl tar vim wget
    done

    for i in master-1 master-2 worker-1 worker-2; do
      ssh $i sudo mkdir /var/lib/kubernetes/
    done
```

- (Optional) setup ssh key authenticate from master-1 to all other servers to transfer files/certs convenientely 
- (Optional) setup ssh key authenticate from master-1 to all other servers to transfer files/certs convenientely 
- (Optional) update /etc/hosts in all  servers as below

```
   for i in master-1 master-2 worker-1 worker-2; do
      ssh $i sudo echo 192.168.1.50 lb >> /etc/hosts
      ssh $i sudo echo 192.168.1.51 master-1 >> /etc/hosts
      ssh $i sudo echo 192.168.1.52 master-2 >> /etc/hosts
      ssh $i sudo echo 192.168.1.53 worker-1 >> /etc/hosts
      ssh $i sudo echo 192.168.1.54 worker-2 >> /etc/hosts
   done
```

- From master-1, create the below directories to organize the files
```
   for i in master-1 master-2; do
      ssh $i sudo mkdir /etc/etcd/
   done
```

- Create the worker nodes directories

```
    for i in worker-1 worker-2; do
        ssh ${i} sudo mkdir -p /etc/cni/net.d/ /opt/cni/bin
    done
```

#1. etcd CA generation

```
    openssl genrsa -out etcd-ca.key 4096
    openssl req -x509 -new -nodes \
      -key etcd-ca.key \
      -sha256 \
      -days 3650 \
      -subj "/CN=etcd-ca" \
      -addext "basicConstraints=critical,CA:TRUE" \
      -addext "keyUsage=critical,keyCertSign,cRLSign" \
      -addext "subjectKeyIdentifier=hash" \
      -addext "authorityKeyIdentifier=keyid:always,issuer" \
      -out etcd-ca.crt
```

#2. kubernetes CA generation
```
   openssl genrsa -out ca.key 2048
   openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
   openssl req -x509 -new -nodes -key ca.key -days 3650 -out ca.crt -subj "/CN=KUBERNETES-CA" -addext "basicConstraints=critical,CA:TRUE" -addext "keyUsage=critical,keyCertSign,cRLSign" -addext "subjectKeyIdentifier=hash"
```

#3. Download kubectl and distribute to all server used for administration

```
    wget https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubectl
    chmod +x kubectl
    for i in master-1 master-2; do
      scp kubectl ${i}:/tmp/
      ssh $i sudo mv /tmp/kubectl /usr/local/bin/
    done
```

#4. Copy of the CA and etcd-CA (e.g. below)
```
   scp ca.crt ca.key etcd-ca.key etcd-ca.crt root@master-1:/var/lib/kubernetes/
   scp ca.crt ca.key etcd-ca.key etcd-ca.crt root@master-2:/var/lib/kubernetes/
   scp ca.crt root@worker-1:/var/lib/kubernetes/
   scp ca.crt root@worker-2:/var/lib/kubernetes/
   scp etcd-ca.key etcd-ca.crt root@master-1:/etc/etcd/
   scp etcd-ca.key etcd-ca.crt root@master-2:/etc/etcd/
```

#These certificates needs to be kept as they will be used to sign future certificates, so basically they will be part of your local Certificate Authority

[Next: Setup etcd](etcd-setup.md)
