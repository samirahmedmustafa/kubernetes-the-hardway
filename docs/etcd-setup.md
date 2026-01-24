# etcd_configuration

- Create and edit the servers openssl configuration files such as mentioned [server1](etcd-server1.cnf) as etcd-server1.cnf and [server2](etcd-server2.cnf) as etcd-server2.cnf

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
  scp etcd-server1.crt etcd-server1.key root@master-1:/etc/etcd
  scp etcd-server2.crt etcd-server2.key root@master-2:/etc/etcd
  ```
- Download etcd, extract it and move it to a standard binaries location

  	```
	wget https://github.com/etcd-io/etcd/releases/download/v3.6.7/etcd-v3.6.7-linux-amd64.tar.gz
  	tar -xzf etcd-v3.6.7-linux-amd64.tar.gz
  	scp etcd-v3.6.7-linux-amd64/etcd* root@master-1:/usr/bin/
  	scp etcd-v3.6.7-linux-amd64/etcd* root@master-2:/usr/bin/
	```
- Create the systemd service files for the 2 servers as mentioned in the links [server1](etcd1.server.service) as etcd_server1.service and [server2](etcd2.server.service) as etcd_server2.service those files should be renamed to etcd.service in the 2 servers
	
	```
	  scp etcd_server1.service master-1:/etc/systemd/system/etcd.service	
	  scp etcd_server2.service master-2:/etc/systemd/system/etcd.service	
	```

- Start etcd services in the 2 servers

	```
 	for i in master-1 master-2; do
 		ssh $i sudo systemctl daemon-reload
		ssh $i sudo systemctl enable --now etcd
 	done
 	```
	```
 	ssh master-1 sudo etcdctl member list   --endpoints=https://127.0.0.1:2379   --cacert=/etc/etcd/etcd-ca.crt   --cert=/etc/etcd/etcd-server1.crt   --key=/etc/etcd/etcd-server1.key
 	```
[Previous: Prerequisites](prerequisites.md)&nbsp;&nbsp;&nbsp;&nbsp;[Next: Setup admin account](admin-account-setup.md)
