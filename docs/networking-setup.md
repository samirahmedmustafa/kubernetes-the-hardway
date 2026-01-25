Download and install cilium binaries for networking (as noted in cilium [website](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/))

```
       CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
       CLI_ARCH=amd64
       if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
       curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
       sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
       mkdir cilium_dir
       tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz cilium_dir
       scp -p cilium_dir/* root@master-1:/usr/local/bin/
```

2. Install cilium

   ```
   ssh master-1 sudo cp admin.kubeconfig ~root/.kube/config
   ssh master-1 sudo /usr/local/bin/cilium install \
         --version 1.18.5 \
         --set ipam.mode=cluster-pool \
         --set ipam.operator.clusterPoolIPv4PodCIDRList={10.0.0.0/16} \
         --set ipam.operator.clusterPoolIPv4MaskSize=24 \
         --set serviceClusterIPRange=10.96.0.0/12 \
         --set kubeProxyReplacement=true \
         --set hubble.enabled=true \
         --set hubble.relay.enabled=true \
         --set hubble.ui.enabled=true \
         --set cluster.name=home-cluster \
         --wait
   ```
[Previous: Setup kubelet and kube-proxy in worker nodes](worker-nodes-setup.md)&nbsp;&nbsp;&nbsp;&nbsp;[Setup coredns](coredns-setup.md)
