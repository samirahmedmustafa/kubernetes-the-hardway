# The below to be executed from the first node worker-1
0. Disable swap

```
       swapoff -a
       sed -ie 's/.*swap.*defaults/#&/' /etc/fstab
       ssh worker-2 swapoff -a
       ssh worker-2 'sed -ie "s/.*swap.*defaults/#&/" /etc/fstab'
```
2. Download kubernetes node binaries, containerd and 
```
    wget -q --show-progress --https-only --timestamping https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubectl \
        https://dl.k8s.io/v1.34.2/bin/linux/amd64/kube-proxy \
        https://dl.k8s.io/v1.34.2/bin/linux/amd64/kubelet \
        https://github.com/containerd/containerd/releases/download/v2.2.1/containerd-2.2.1-linux-amd64.tar.gz
        
```

```
    tar -xzf containerd-2.2.1-linux-amd64.tar.gz
    install bin/* /usr/local/bin/
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

4. Create kubelet config.yaml

```
cat > config.yaml <<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.crt"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.96.0.10"
resolvConf: "/etc/resolv_k8s.conf"
runtimeRequestTimeout: "15m"
EOF
```

```
    for i in worker-1 worker-2; do
        scp config.yaml ${i}:/var/lib/kubernetes/
    done
```
5. Create kubelet systemd service file

```
cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --kubeconfig=/var/lib/kubernetes/kubeconfig \\
  --bootstrap-kubeconfig=/var/lib/kubernetes/bootstrap-kubeconfig \\
  --config=/var/lib/kubernetes/config.yaml \\
  --rotate-certificates=true \\
  --rotate-server-certificates=true \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
    for i in worker-1 worker-2; do
        scp kubelet.service ${i}:/etc/systemd/system/
    done
```
5. Download and deploy CNI

```
    wget https://github.com/containernetworking/plugins/releases/download/v1.9.0/cni-plugins-linux-amd64-v1.9.0.tgz
    tar -xzvf cni-plugins-linux-amd64-v1.9.0.tgz -C /opt/cni/bin/
    rm -f cni-plugins-linux-amd64-v1.9.0.tgz
```

6. Deploy containerd systemd service file

```
cat > /etc/systemd/system/containerd.service <<EOF
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target dbus.service

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF
```

```
    scp /etc/systemd/system/containerd.service worker-2:/etc/systemd/system/
```

```
       systemctl daemon-reload
       systemctl enable --now kubelet
       ssh worker-2 systemctl daemon-reload
       ssh worker-2 systemctl enable --now kubelet
```
7. Download and install cilium binaries for networking (as noted in cilium [website](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/))
```
    CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
    CLI_ARCH=amd64
    if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
    curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
    sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
    sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
    rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```
