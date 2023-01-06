---
title: "Generate Self-Signed CA"
date: 2023-01-06
last_modified_at: 2023-01-06
categories:
  - blog
tags:
  - CA
---

In bookstore project, I tried to enable https protocol on my spring server. I didn't find any tutorial to correctly generate and import CA, so I write this blog to record my solution.

## Steps

1. Generate key-pair for CA Root
2. Generate `Certificate` of CA Root
3. Generate `Sign Request`
4. Use CA Root's key-pair to sign `Sign Request`
5. Import `CA Root`'s `CA` into system

---

Here is the code. I use `openssl` to generate key for `CA Root`, and `keytool` for my bookstore server.

```bash
mkdir ca # create dir for CA files

openssl genrsa -out ca/ca-key.pem 2048 # generate CA Root keypair
openssl req -new -out ca/ca-req.csr -key ca/ca-key.pem # generate CA Root Certificate Sign Request
openssl x509 -req -in ca/ca-req.csr -signkey ca/ca-key.pem -out ca/ca-cert.pem # sign CSR to generate CA Root Certificate
sudo cp ca/ca-cert.pem /etc/pki/ca-trust/source/anchors/ca-cert.pem # Import CA Root Certificate to system
sudo update-ca-trust # Update CA Trust List

keytool -genkey -alias example -validity 365 -keyalg RSA -keypass 123456 -storepass 123456 -keystore example.jks -ext SAN=dns:localhost,ip:127.0.0.1 -rfc -dname "CN=localhost, OU=Server, O=SJTU, L=Shanghai, S=Shanghai, C=US" # generate server keypair
keytool -certreq -alias example -file example.csr -keypass 123456 -keystore example.jks -storepass 123456 -ext SAN=dns:localhost,ip:127.0.0.1 # generate Certificate Sign Request File for server keypair 


openssl x509 -req -in example.csr -out example.pem -CA ca/ca-cert.pem -CAkey ca/ca-key.pem -days 365 -set_serial 1  -extfile openssl.cnf  -extensions v3_req # sign CSR to generate server Certificate

keytool -import -v -alias root -keypass 123456 -trustcacerts -file ca/ca-cert.pem -storepass 123456 -keystore example.jks # import CA Root Certificate into server keystore
keytool -import -v -alias example -trustcacerts -file example.pem -storepass 123456 -keystore example.jks # import server Certificate into server keystore
```
