# How CA Actually Works

1. You generate your public and private key pair.
2. You create a Certificate Signing Request (CSR): This includes your public key and information about your identity (e.g., your domain name for a website, your organization's name).
3. You send the CSR to a Certificate Authority (CA): The CA is a trusted third-party organization.
4. The CA verifies your identity: The CA goes through a process (which varies based on the type of certificate) to ensure you are who you claim to be.
5. The CA creates a digital certificate: If your identity is verified, the CA takes your public key and your identity information and digitally signs this bundle using its own private key. This signed bundle is your digital certificate.
6. You use the digital certificate: When someone wants to communicate securely with you (e.g., their web browser connecting to your website), you present your digital certificate.
7. Verification by the user:
    - The user's software (e.g., web browser) receives your certificate.
    - It retrieves the CA's public key (which it usually already has because popular CA public keys are pre-installed as "trusted root certificates" in browsers and operating systems).
    - It uses the CA's public key to verify the digital signature on your certificate.
    - If the signature is valid, the browser trusts that your public key (contained within the certificate) is authentic and indeed belongs to you, as vouched for by the trusted CA.

In summary, your core understanding of the "why" (trusting public keys) and the "who" (Certificate Authorities) is excellent. The main correction is in the mechanics of how the CA processes your public key – it signs it, rather than encrypts it, and the user verifies that signature with the CA's public key.

---

## Core Concepts and File Formats

- `.key` (Private Key)
- `.csr` (Certificate Signing Request)
- `.crt` (Certificate) / `.pem` (Certificate - often a chain)
- `openssl.cnf` (OpenSSL Configuration File)

---

## Server 1 (Master-eligible, Data, Kibana)

- Hostname: `master-worker1`
- IP Address: `192.168.1.14`

---

## Step 1: CA Setup (Directory and openssl.cnf)

```bash
# On your CA machine (e.g., master-worker1, or a separate admin host)
mkdir my-es-cluster-ca
cd my-es-cluster-ca

mkdir certs newcerts private csr  # Directories for issued certs, new certs, private keys, and CSRs
touch index.txt                   # CA database of issued certificates
echo 1000 > serial               # Starting serial number for your certificates
```

### Now, create the openssl.cnf file

```ini
# This is the OpenSSL configuration file for your custom CA.

[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = ./
certs             = $dir/certs
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
crldir            = $dir/crl
crl               = $crldir/crl.pem
private_key       = $dir/private/ca-key.pem
certificate       = $dir/ca-cert.pem
policy            = policy_strict
RANDFILE          = $dir/.rand

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 3650
default_crl_days  = 30
default_md        = sha256
preserve          = no
email_in_dn       = no
batch             = no
unique_subject    = no

[ policy_strict ]
countryName             = match
stateOrProvinceName     = optional
organizationName        = match
organizationalUnitName  = optional
commonName              = match
emailAddress            = optional

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = CA:TRUE
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_req_es_node ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names_es_node

[ alt_names_es_node ]
DNS.1 = master-worker1
DNS.2 = worker2
DNS.3 = worker3
IP.1 = 192.168.1.14
IP.2 = 192.168.1.13
IP.3 = 192.168.1.12
DNS.4 = localhost
IP.4 = 127.0.0.1

[ v3_req_kibana ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names_kibana

[ alt_names_kibana ]
DNS.1 = master-worker1
IP.1 = 192.168.1.14
DNS.2 = localhost
IP.2 = 127.0.0.1

[ req ]
default_bits        = 4096
default_md          = sha256
prompt              = yes
distinguished_name  = req_dn
x509_extensions     = v3_ca

[ req_dn ]
countryName                     = Country Name (2 letter code)
countryName_default             = IR
stateOrProvinceName             = State or Province Name
stateOrProvinceName_default     = Tehran
organizationName                = Organization Name
organizationName_default        = aip
commonName                      = Common Name
commonName_default             = My Root CA
emailAddress                    = Email Address
emailAddress_default            = en.habibi1377@gmail.com
```

---

### Key Points in openssl.cnf

- `[alt_names_es_node]` now contains all hostnames and IP addresses of all your Elasticsearch nodes.  
- `[alt_names_kibana]` contains only the names/IPs specific to the Kibana server (`master-worker1` in this case).
- `extendedKeyUsage = serverAuth, clientAuth` for `v3_req_es_node` is vital for Elasticsearch nodes.

---

## Step 2: Create the CA's Private Key and Certificate

```bash
# Still in my-es-cluster-ca/ directory

# Generate a 4096-bit RSA private key, encrypted with AES-256
openssl genrsa -aes256 -out private/ca-key.pem 4096

# Set strong file permissions
chmod 400 private/ca-key.pem

# Create the CA's self-signed certificate
openssl req -config openssl.cnf     -new -x509 -days 3650     -key private/ca-key.pem     -sha256 -extensions v3_ca     -out ca-cert.pem
```

---

## Step 3: Generate Certificates for Elasticsearch Nodes

For `master-worker1`:

```bash
# Still in my-es-cluster-ca/ directory

# 3.1. Create Private Key for master-worker1
openssl genrsa -out private/master-worker1.key 2048
chmod 400 private/master-worker1.key

# 3.2. Create CSR for master-worker1
openssl req -config openssl.cnf     -new -key private/master-worker1.key     -sha256     -reqexts v3_req_es_node     -out csr/master-worker1.csr

# 3.3. Sign the CSR for master-worker1 with your CA
openssl ca -config openssl.cnf     -extensions v3_req_es_node     -days 1095     -notext -md sha256     -in csr/master-worker1.csr     -out certs/master-worker1.crt
```

---

## Step 4: Generate Certificate for Kibana

```bash
# Still in my-es-cluster-ca/ directory

# 4.1. Create Private Key for Kibana
openssl genrsa -out private/kibana.key 2048
chmod 400 private/kibana.key

# 4.2. Create CSR for Kibana
openssl req -config openssl.cnf     -new -key private/kibana.key     -sha256     -reqexts v3_req_kibana     -out csr/kibana.csr

# 4.3. Sign the CSR for Kibana with your CA
openssl ca -config openssl.cnf     -extensions v3_req_kibana     -days 1095     -notext -md sha256     -in csr/kibana.csr     -out certs/kibana.crt
```

---

## Step 5: Prepare PKCS#12 Keystores (Recommended for Elasticsearch)

For `master-worker1`:

```bash
openssl pkcs12 -export     -in certs/master-worker1.crt     -inkey private/master-worker1.key     -certfile ca-cert.pem     -out certs/master-worker1.p12     -name "master-worker1-cert"
```
in elastic directory like /home/$user/els1/config/elasticsearch.yml put your certifacate path and password 
```bash
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
# Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
xpack.security.http.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/master-worker1.p12
  keystore.password: "Necessary87215"
  certificate_authorities: [ "/home/konect/els1/config/certs/ca-cert.pem" ]
# Enable encryption and mutual authentication between cluster nodes
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/master-worker1.p12
  keystore.password: "Necessary87215"
  certificate_authorities: [ "/home/konect/els1/config/certs/ca-cert.pem" ]
```

You can use this execute file to introduce `.p12` password to Elasticsearch:

```bash
bin/elasticsearch-keystore add     xpack.security.transport.ssl.keystore.secure_password
```
