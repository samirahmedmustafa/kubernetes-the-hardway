- Download the coredns manifest

```
  git clone https://github.com/coredns/deployment.git
  cd deployment/kubernetes/
  cp deployment/kubernetes/coredns.yaml.sed ./coredns.yaml
  sed -ie 's/CLUSTER_DNS_IP/10.96.0.10/' coredns.yaml
```

2. Apply the coredns
```
  kubectl apply -f coredns.yaml
```
