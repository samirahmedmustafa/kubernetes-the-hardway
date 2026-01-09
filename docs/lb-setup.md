1. Install haproxy

```
    ssh lb dnf install -y -q haproxy vim
```

2. Configure the haproxy

```
cat <<EOF | tee haproxy.cfg 
frontend kubernetes
    bind 192.168.1.50:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server master-1 192.168.1.51:6443 check fall 3 rise 2
    server master-2 192.168.1.52:6443 check fall 3 rise 2
EOF

scp haproxy.cfg lb:/etc/haproxy/haproxy.cfg
rm -f haproxy.cfg
```

3. Restart haproxy services
```
    ssh lb systemctl restart haproxy
    ssh lb systemctl enable haproxy
```

[Previous: Setup kube-apiserver](kube-apiserver-setup.md)&nbsp;&nbsp;&nbsp;&nbsp;[Setup kubei-apiserver](kube-apiserver-setup.md)

