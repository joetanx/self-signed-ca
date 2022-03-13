# Generate a self-signed certificate chain with openssl

## Example used for this guide: Conjur
This guide is applicable for both Conjur Open Source Suite (OSS) and Enterprise Edition

### Conjur Open Source Suite
- Conjur OSS relies on the nginx proxy container to provide communication over TLS
- The default method to deploy Conjur OSS using quick start includes an openssl container in the Docker Compose manifest
  - Read: <https://github.com/cyberark/conjur-quickstart>
- In my method of deploying a minimal Conjur OSS using `podman play kube`, you can generate the certificate using openssl, then put it in a volume mount to `/etc/nginx/tls` of the nginx container
  - Read: <https://joetanx.github.com/conjur-oss>

### Conjur Enterprise Edition
- Conjur EE uses certificates for communication between the Master, Standby, and follower nodes in Conjur cluster.
- To understand Conjur certificate architecture, read: <https://docs.cyberark.com/Product-Doc/OnlineHelp/AAM-DAP/Latest/en/Content/Deployment/HighAvailability/certificate-architecture.htm>

### Certificate to be generated in this guide

| Role | Common Name | Subject Alternative Name |
| --- | --- | --- |
| Certificate Authority | vx Lab Certificate Authority | |
| Conjur Server | conjur.vx | conjur.vx |
| Conjur Follower | follower.conjur.svc.cluster.local | follower.conjur.svc.cluster.local |

## 1.0 Generate a self-signed certificate authority
### 1.1 Method 1: Generate key first, then CSR, then certificate
- Generate private key of the self-signed certificate authority
```console
[root@conjur ~]# openssl genrsa -out vxLabCA.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
...........................................................................+++++
.......................................+++++
e is 65537 (0x010001)
```
- Generate certificate of the self-signed certificate authority
- **Note**: change the common name of the certificate according to your environment
```console
[root@conjur ~]# openssl req -x509 -new -nodes -key vxLabCA.key -days 365 -sha256 -out vxLabCA.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:.
State or Province Name (full name) []:
Locality Name (eg, city) [Default City]:.
Organization Name (eg, company) [Default Company Ltd]:.
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:vx Lab Certificate Authority
Email Address []:
```

### 1.2 Method 2: Generate key and certificate in a single command
```console
[root@conjur ~]# openssl req -newkey rsa:2048 -days "365" -nodes -x509 -keyout vxLabCA.key -out vxLabCA.pem
Generating a RSA private key
...............................................+++++
.........+++++
writing new private key to 'vxLabCA.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:.
State or Province Name (full name) []:
Locality Name (eg, city) [Default City]:.
Organization Name (eg, company) [Default Company Ltd]:.
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:vx Lab Certificate Authority
Email Address []:
```

## 2.0 Generate certificate for Conjur Server
> **Note**: change the common name/subject alternative name of the certificate according to your environment
> 
> The name must match the Conjur Server FQDN that the clients or followers will be using to communicate to the Conjur Server
> 
> If your Conjur Enterprise deployment includes master/standby setup, you need to include the FQDNs of the load balancer, master server, and standby servers in the certificate SAN

- Generate **private key** of the Conjur Server certificate
```console
openssl genrsa -out conjur.vx.key 2048
```
- Create **certificate signing request** for the Conjur Master certificate
```console
openssl req -new -key conjur.vx.key -subj "/CN=conjur.vx" -out conjur.vx.csr
```
- Create OpenSSL **configuration file** to add subject alternative name
```console
echo "subjectAltName=DNS:conjur.vx" > conjur.vx-openssl.cnf
```
- Generate **certificate** of the Conjur Master certificate
```console
openssl x509 -req -in conjur.vx.csr -CA vxLabCA.pem -CAkey vxLabCA.key -CAcreateserial -days 365 -sha256 -out conjur.vx.pem -extfile conjur.vx-openssl.cnf
```

## 3.0 Generate certificate for Conjur Follower
> **Note**: change the common name/subject alternative name of the certificate according to your environment
> 
> The name must match the follower FQDN that the follower will be using to communicate to the Conjur Master
> 
> For follower deployment in Kubernetes, the name will be the Kubernetes service FQDN in the form of `<service-name>.<namespace>.svc.cluster.local`
> 
> Conjur OSS does not support followers

- Generate **private key** of the Conjur Follower certificate
```console
openssl genrsa -out follower.conjur.svc.cluster.local.key 2048
```
- Create **certificate signing request** for the Conjur Follower certificate
```console
openssl req -new -key follower.conjur.svc.cluster.local.key -subj "/CN=follower.conjur.svc.cluster.local" -out follower.conjur.svc.cluster.local.csr
```
- Create OpenSSL **configuration file** to add subject alternative name
```console
echo "subjectAltName=DNS:follower.conjur.demo,DNS:follower.conjur.svc.cluster.local" > follower.conjur.svc.cluster.local-openssl.cnf
```
- Generate **certificate** of the Conjur Follower certificate
```console
openssl x509 -req -in follower.conjur.svc.cluster.local.csr -CA ConjurDemoCA.pem -CAkey ConjurDemoCA.key -CAcreateserial -days 365 -sha256 -out follower.conjur.svc.cluster.local.pem -extfile follower.conjur.svc.cluster.local-openssl.cnf
```
