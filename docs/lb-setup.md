1. Install haproxy

```
    ssh lb sudo dnf install -y -q haproxy vim
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

scp haproxy.cfg root@lb:/etc/haproxy/haproxy.cfg
```

3. Allow 6443 port in the loadbalancer

```
    ssh lb sudo firewall-cmd --add-port=6443/tcp
    ssh lb sudo firewall-cmd --permanent --add-port=6443/tcp
```

4. Restart haproxy services
```
    ssh lb sudo systemctl restart haproxy
    ssh lb sudo systemctl enable haproxy
```

[Previous: Setup service account](service-account-setup.md)&nbsp;&nbsp;&nbsp;&nbsp;[Setup kube-apiserver](kube-apiserver-setup.md)

