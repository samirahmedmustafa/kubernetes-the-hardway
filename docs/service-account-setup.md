#Create service account to be used for signing new user accounts (execute the below inside the CA directory)

1. Create the certificates
```
    openssl genrsa -out service-account.key 2048
    openssl req -new -key service-account.key -subj "/CN=service-accounts" -out service-account.csr
    openssl x509 -req -in service-account.csr -CA /var/lib/kubernetes/ca.crt -CAkey /var/lib/kubernetes/ca.key -CAcreateserial  -out service-account.crt -days 1000
```

```
    openssl verify -CAfile /var/lib/kubernetes/ca.crt service-account.crt
```
2. Copy the certificates to the kubernetes location

```
    cp service-account.crt service-account.key /var/lib/kubernetes/
    scp service-account.crt service-account.key master-2:/var/lib/kubernetes/
```

[Previous: Setup admin account](admin-account-setup.md)&nbsp;&nbsp;&nbsp;&nbsp;[Next: Setup kube-apiserver](kube-apiserver-setup.md)

