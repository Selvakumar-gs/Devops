OpenSSL is a free, open-source library that you can use for digital certificates. One of the things you can do is build your own CA (Certificate Authority).

Instead of paying companies like Verisign for all your digital certificates. It can be useful to build your own CA for some of your applications. In this lesson, you will learn how to create your own CA.

Configuration
In my examples, I will use a Ubuntu server, the configuration of openSSL will be similar though on other distributions like CentOS.

Prerequisites
Before we configure OpenSSL, I like to configure the hostname/FQDN correctly and make sure that our time, date and timezone is correct.
Let’s take a look at the hostname:
++++++++++++++++++
# hostname
++++++++++++++++++
Let’s check the FQDN:
++++++++++++++++++
# hostname -f
++++++++++++++++++
Add host entry >> 
vim /etc/hosts
10.192.190.152 devxptai.ardianet.net
10.192.190.152 devxptai-revamp.ardianet.net

++++++++++++++++++++++++++++++++++++++++++++++++++++++

Create a conf file 
ca_cert.cnf
server_cert.cnf
server_ext.cnf
**bottom i have exact file**

Generate Root CA Certificate
The Root Certificate Authority (CA) certificate is the foundation of your certificate chain. It will sign your server certificate.

Step 1: Generate Root CA Private Key
Generate a 4096-bit RSA private key for the Root CA.
openssl genrsa -out ./dev-certs/ca.key 4096|

Step 2: Generate Root CA Self-Signed Certificate
Generate the Root CA certificate, which is self-signed and valid for 10 years (3650 days).
openssl req -new -x509 -days 3650 -config ./ca_cert.cnf -key ./dev-certs/ca.key -out ./dev-certs/cacert.pem

Step 3: Verify Root CA Certificate
Verify that the Root CA certificate is correctly generated and readable.
openssl x509 -noout -text -in ./dev-certs/cacert.pem

Generate Server Certificate
The server certificate will be signed by the Root CA and can be used for secure communications with clients.

Step 1: Generate Server Private Key
Generate a 4096-bit RSA private key for the server.
openssl genrsa -out ./dev-certs/server.key 4096”

Step 2: Generate Server Certificate Signing Request (CSR)
Generate the server's CSR, which will be signed by the Root CA.
openssl req -new -key ./dev-certs/server.key -out ./dev-certs/server.csr -config ./server_cert.cnf

Step 3: Generate Signed Server Certificate
Sign the server's CSR with the Root CA certificate to generate the server certificate, which is valid for 10 years (3650 days).
openssl x509 -req -in ./dev-certs/server.csr -CA ./dev-certs/cacert.pem -CAkey ./dev-certs/ca.key -out ./dev-certs/server.crt -CAcreateserial -days 3650 -sha512 -extfile ./server_ext.cnf

Step 4: Verify the Server Certificate
Verify that the server certificate is valid and correctly signed by the Root CA.
openssl verify -CAfile ./dev-certs/cacert.pem ./dev-certs/server.crt

Setup SSL in kubernetes pod with nginx ingress

First install the nginx-ingress in kubernetes cluster

kubectl create secret tls devxptai-revamp-ssl --namespace devxptai --key server.key --cert server.crt

•	It is recommended to deploy the NGINX Ingress Controller within its own namespace.
   kubectl create namespace ingress-nginx
•	Getting the latest version of nginx ingress
 CONTROLLER_TAG=$(curl -s https://api.github.com/repos/kubernetes/ingress-nginx/releases/latest | grep tag_name | cut -d '"' -f 4)

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/${CONTROLLER_TAG}/deploy/static/provider/baremetal/deploy.yaml
•	Verify the Installation.
Check if the NGINX Ingress Controller pods are running in the specified namespace:
    kubectl get pods -n ingress-nginx



cat <<EOF >  ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-ingress
  namespace: dev
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - <FQDN>
    secretName: <SSLSECRET NAME>
  rules:
  - host: "FQDN"
    http:
      paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: SVC_NAME
              port:
                number: 80
EOF


+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++


Creating ca_cert.cnf file
cat <<EOF > ca_cert.cnf
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_ca
prompt = no

[req_distinguished_name]
countryName             = IN
stateOrProvinceName     = TN
localityName            = Chennai
organizationName        = Dev
commonName              = Root_CA

[ v3_ca ]
basicConstraints=critical,CA:TRUE
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer 

EOF

++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Creating Server_cert.cnf file
cat <<EOF > server_cert.cnf
[req]
default_bit = 4096
distinguished_name = req_distinguished_name
prompt = no

[req_distinguished_name]
countryName             = IN
stateOrProvinceName     = TN
localityName            = Chennai
organizationName        = dev
commonName              = dev.example.com
 
EOF 

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Creating Server_ext.cnf file

cat <<EOF > server_ext.cnf
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[alt_names]
IP.1 = 10.192.190.152
DNS.1 = dev.example.com 

EOF
