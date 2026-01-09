# The below to be executed from the first node worker-1

1. Download kubernetes node binaries, containerd and 
```
    wget -q --show-progress --https-only --timestamping https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubectl \
        https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-proxy \
        ttps://dl.k8s.io/v1.34.2/bin/linux/amd64/kubelet \
        https://github.com/containerd/containerd/releases/download/v2.2.1/containerd-2.2.1-linux-amd64.tar.gz \
        
```

2. Create directories
```
    mkdir -p /var/lib/kubernetes/ /etc/cni/net.d /opt/cni/bin/
    ssh worker-2 mkdir -p /etc/cni/net.d /opt/cni/bin /var/lib/kubernetes/
```

3. Change binaries to execution in the 2 worker nodes
```
    chmod +x kubectl kube-proxy kubelet
    install kubectl kube-proxy kubelet /usr/local/bin/
    scp /usr/local/bin/kube* worker-2:/usr/local/bin/
    ssh worker-2 chmod +x /usr/local/bin/kube*
```

4. Create a bootstrap kubeconfig

```
vim /var/lib/kubelet/bootstrap-kubeconfig

apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /var/lib/kubernetes/ca.crt
    server: https://192.168.1.50:6443
  name: bootstrap
contexts:
- context:
    cluster: bootstrap
    user: kubelet-bootstrap
  name: bootstrap
current-context: bootstrap
preferences: {}
users:
- name: kubelet-bootstrap
  user:    02b50b.05283e98dd0fd71db496ef01e8
    token: 07401b.f395accd246ae52d

```

5. Download and deploy CNI

```
    wget https://github.com/containernetworking/plugins/releases/download/v1.9.0/cni-plugins-linux-amd64-v1.9.0.tgz
    tar -xzvf cni-plugins-linux-amd64-v1.9.0.tgz --directory /opt/cni/bin/
    rm -f cni-plugins-linux-amd64-v1.9.0.tgz
```

6. Download and install cilium binaries for networking (as noted in cilium [website](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/))
```
    CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
    CLI_ARCH=amd64
    if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
    curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
    sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
    sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
    rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```
