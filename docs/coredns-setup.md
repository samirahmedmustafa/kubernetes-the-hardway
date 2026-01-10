1. Check whether the certificate are approved or not
```
  kubectl get csr
  #approve them if not
  kubectl get csr -o name | xargs kubectl certificate approve
```

2. Download the coredns manifest

```
  git clone https://github.com/coredns/deployment.git
  cd deployment/kubernetes/
  cp deployment/kubernetes/coredns.yaml.sed ./coredns.yaml
  sed -ie 's/CLUSTER_DNS_IP/10.96.0.10/' coredns.yaml
```

3. Apply the coredns
```
  kubectl apply -f coredns.yaml
```

4. Crash issues

   - plugin/forward: no nameservers found
     ```
     kubectl -n kube-system edit configmap coredns
     #modify forward . /etc/resolv.conf to forward . 8.8.8.8 1.1.1.1
     ```

[Setup (cilium) networking](networking-setup.md)
