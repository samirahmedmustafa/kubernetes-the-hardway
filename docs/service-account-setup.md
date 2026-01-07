#Create service account to be used for signing new user accounts (execute the below inside the CA directory)

```
    openssl genrsa -out service-account.key 2048
    openssl req -new -key service-account.key -subj "/CN=service-accounts" -out service-account.csr
    openssl x509 -req -in service-account.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out service-account.crt -days 1000
```

