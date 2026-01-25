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
```
- Edit coredns.yaml, replace the ConfigMap with the below
   
replace
```
grep -B1 -A23 ConfigMap coredns.yaml

apiVersion: v1
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
apiVersion: v1
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
  scp coredns.yaml master-1:
  ssh master-1 sudo /usr/local/bin/kubectl apply -f coredns.yaml
```


[Setup (cilium) networking](networking-setup.md)







