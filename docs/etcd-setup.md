# etcd_configuration

- Generate the CA using the below command (pay attention to CA:TRUE)

	```
	  openssl genrsa -out etcd-ca.key 4096
	  openssl req -x509 -new -nodes \
	    -key etcd-ca.key \
	    -sha256 \
	    -days 3650 \
	    -subj "/CN=etcd-ca" \
	    -addext "basicConstraints=critical,CA:TRUE" \
	    -addext "keyUsage=critical,keyCertSign,cRLSign" \
	    -addext "subjectKeyIdentifier=hash" \
	    -addext "authorityKeyIdentifier=keyid:always,issuer" \
	    -out etcd-ca.crt
	```
- Create and edit the servers openssl configuration files such as mentioned [server1](etcd-server1.cnf) and [server2](etcd-server2.cnf)

- Generate the server certificates and sign them as below

	```
	  openssl genrsa -out etcd-server1.key 2048
	  openssl req -new -key etcd-server1.key -out etcd-server1.csr -config etcd-server1.cnf
	  openssl x509 -req -in etcd-server1.csr -CA etcd-ca.crt -CAkey etcd-ca.key -CAcreateserial -out etcd-server1.crt -days 1000 -extensions v3_req -extfile etcd-server1.cnf
	```
	```
	  openssl genrsa -out etcd-server2.key 2048
	  openssl req -new -key etcd-server2.key -out etcd-server2.csr -config etcd-server2.cnf
	  openssl x509 -req -in etcd-server2.csr -CA etcd-ca.crt -CAkey etcd-ca.key -CAcreateserial -out etcd-server2.crt -days 1000 -extensions v3_req -extfile etcd-server2.cnf
	```
- Verify

	```
	  openssl verify -CAfile etcd-ca.crt etcd-server1.crt && openssl verify -CAfile etcd-ca.crt etcd-server2.crt
	  openssl x509 -in etcd-server1.crt -noout -text | grep -A2 "Extended Key Usage"
	  openssl x509 -in etcd-server2.crt -noout -text | grep -A2 "Extended Key Usage"
			  
	```
- Copy certificates to the /etc/etcd in both servers
  ```
  mkdir /etc/etcd
  cp etcd-server1.crt etcd-server1.key etcd-ca.crt /etc/etcd
  ssh master-2 mkdir /etc/etcd
  scp etcd-server2.crt etcd-server2.key etcd-ca.crt master-2:/etc/etcd  
  ```
- Download etcd, extract it and move it to a standard binaries location

  	```
	  wget https://github.com/etcd-io/etcd/releases/download/v3.6.7/etcd-v3.6.7-linux-amd64.tar.gz
  	  tar -xzf etcd-v3.6.7-linux-amd64.tar.gz
  	  cp etcd-v3.6.7-linux-amd64/etcd* /usr/bin/

	```
- Create the systemd service files for the 2 servers as mentioned in the links [server1](etcd1.server.service) and [server2](etcd2.server.service) those files should be renamed to etcd.service in the 2 servers
	
	```
	  cp etcd1.server.service /etc/systemd/system/etcd.service	
	  scp etcd2.server.service master-2:/etc/systemd/system/etcd.service	
	```

- Start etcd services in the 2 servers

	```
 	systemctl daemon-reload
	systemctl enable --now etcd
 	ssh master-2 systemctl daemon-reload
	ssh master-2 systemctl enable --now etcd
	etcdctl member list   --endpoints=https://127.0.0.1:2379   --cacert=/etc/etcd/etcd-ca.crt   --cert=/etc/etcd/etcd-server1.crt   --key=/etc/etcd/etcd-server1.key
 	```

 [Previous: Prerequisities](prerequisites.md) [Next: Setup admin account](admin-account-setup.md)
