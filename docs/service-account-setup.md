#Create service account to be used for signing new user accounts (execute the below inside the CA directory)

1. Create the certificates
```
    openssl genrsa -out service-account.key 2048
    openssl req -new -key service-account.key -subj "/CN=service-accounts" -out service-account.csr
    openssl x509 -req -in service-account.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out service-account.crt -days 1000
```

```
    openssl verify -CAfile ca.crt service-account.crt
```
2. Copy the certificates to the kubernetes location

```
    scp service-account.crt service-account.key root@master-1:/var/lib/kubernetes/
    scp service-account.crt service-account.key root@master-2:/var/lib/kubernetes/
```

[Previous: Setup admin account](admin-account-setup.md)&nbsp;&nbsp;&nbsp;&nbsp;[Next: Setup haproxy loadbalancer](lb-setup.md)

