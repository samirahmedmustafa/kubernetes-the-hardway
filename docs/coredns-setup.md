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

3. Apply the coredns
```
  ssh master-1 sudo /usr/local/bin/kubectl apply -f coredns.yaml
```

4. Crash issues

   - plugin/forward: no nameservers found
     ```
     kubectl -n kube-system edit configmap coredns
     #modify forward . /etc/resolv.conf to forward . 8.8.8.8 1.1.1.1
     ```

[Setup (cilium) networking](networking-setup.md)
