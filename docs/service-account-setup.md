#Create service account to be used for signing new user accounts (execute the below inside the CA directory)

```
    openssl genrsa -out service-account.key 2048
    openssl req -new -key service-account.key -subj "/CN=service-accounts" -out service-account.csr
    openssl x509 -req -in service-account.csr -CA ${ca_crt} -CAkey ${ca_key} -CAcreateserial  -out service-account.crt -days 1000
```

[Previous: Setup admin account](admin-account-setup.md) [Next: setup kube-scheduler](kube-scheduler-setup.md)
