Download and install cilium binaries for networking (as noted in cilium [website](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/))

```
       CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
       CLI_ARCH=amd64
       if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
       curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
       sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
       mkdir cilium_dir
       tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz cilium_dir
       scp -p cilium_dir/* root@master-1/usr/local/bin/
```

2. Install cilium

   ```
   ssh master-1 sudo cilium install --version 1.18.5
   ssh master-1 sudo cilium hubble enable --ui
   ```
[Previous: Setup kubelet and kube-proxy in worker nodes](worker-nodes-setup.md)&nbsp;&nbsp;&nbsp;&nbsp;[Setup coredns](coredns-setup.md)
