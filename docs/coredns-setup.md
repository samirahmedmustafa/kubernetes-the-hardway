1. Check whether the certificate are approved or not
```
  kubectl get csr
  #approve them if not
  ssh master-1 'sudo /usr/local/bin/kubectl get csr -o name | xargs sudo /usr/local/bin/kubectl certificate approve'
```

2. Download the coredns manifest

```
  git clone https://github.com/coredns/deployment.git
  sed -e 's/CLUSTER_DNS_IP/10.96.0.10/' deployment/kubernetes/coredns.yaml.sed > coredns.yaml
  scp coredns.yaml master-1:
```
3. Edit coredns.yaml, replace the ConfigMap with the below
   
replace
  ```
grep -A23 ConfigMap coredns.yaml

kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes CLUSTER_DOMAIN REVERSE_CIDRS {
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . UPSTREAMNAMESERVER {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }STUBDOMAINS
  ```
with the below
  ```
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 8.8.8.8 1.1.1.1 {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
  ```
3. Apply the coredns
```
  ssh master-1 sudo /usr/local/bin/kubectl apply -f coredns.yaml
```

4. Crash issues
```
kubectl -n kube-system logs coredns-769f759fcc-jxwfz
Error from server (Forbidden): Forbidden (user=kube-apiserver, verb=get, resource=nodes, subresource(s)=[proxy])
```
   - plugin/forward: no nameservers found
```
  kubectl -n kube-system edit configmap coredns
  #modify forward . /etc/resolv.conf to forward . 8.8.8.8 1.1.1.1
```

[Setup (cilium) networking](networking-setup.md)



