# The below to be executed from the first node worker-1

1. Download kubernetes node binaries
```
    wget -q --show-progress --https-only --timestamping https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubectl https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-proxy https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubelet
```
2. Create directories
```
    mkdir -p /var/lib/kubernetes/ /etc/cni/net.d /opt/cni/bin/
    ssh worker-2 mkdir -p /etc/cni/net.d /opt/cni/bin /var/lib/kubernetes/
```

3. Change binaries to execution in the 2 worker nodes
```
    chmod +x kubectl kube-proxy kubelet
    mv kubectl kube-proxy kubelet /usr/local/bin/
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
