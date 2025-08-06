+++
date = '2025-07-26T13:35:12+08:00'
draft = false
title = 'PKI Infra Notes'
tags = ['pki']
toc = true
+++

## My Opinions

**Which tool should I use for home lab?**

I personally recommend GnuTLS certtool or smallstep's step\(-ca\).
OpenSSL, the de facto standard, is much more complex and harder to deal with.
Since we're at home, not battlefield, certtool and step may be better options.

For OpenSSL, take a glance at this tutorial first: [OpenSSL PKI Tutorial](https://pki-tutorial.readthedocs.io/en/latest/).

For GnuTLS certtool, remember to attach the man page when talking to LLMs.
The man page also describes the config file in a concise format.

For step\(-ca\), read their documents.

## Create a root CA

### OpenSSL

Create directory structure for the CA components:

```bash
mkdir ca
cd ca
mkdir private certs crl newcerts

touch index.txt
echo 1000 > serial
echo 1000 > crlnumber
```

- `index.txt`: A database file OpenSSL uses to keep track of issued certificates.
- `serial`: Stores the next serial number to be used for certs.
- `crlnumber`: Stores the next CRL number.

Next, create the configuration file for OpenSSL:

```bash
openssl version -d
# OPENSSLDIR: "/etc/ssl"
cp /etc/ssl/openssl.cnf ./root_ca_openssl.cnf
```

Find the "CA_default" region, and set the paths.

```ini
[ CA_default ]
# relative to the cwd
dir = .
# ...
certificate = $dir/certs/ca.crt.pem
private_key = $dir/private/ca.key.pem
```

Next, setup the "v3_ca" extensions for the root CA.
You should see its definition in the "req" section like:

```ini
[ req ]
x509_extensions = v3_ca
# ...
[ v3_ca ]
basicConstraints = critical,CA:true
keyUsage = critical, cRLSign, keyCertSign
```

- `cRLSign`: Allows the CA to sign CRLs.
- `keyCertSign`: Allows the CA to sign other certificates.

> Some people say that root CA should also have "digitalSignature" keyUsage.
> However, Both ISRG root and GTS root did not set this keyUsage, so I'm not setting it either.

> A CA should not set any extended key usage(EKU).

{{< details summary="Full openssl.cnf example (root CA)" file="assets/posts/pki.d/root_ca_openssl.cnf.md" >}}
{{< /details >}}

Generate the private key then.
We'll use the `prime256v1` (NIST P-256) curve here.

```bash
openssl genpkey \
  -algorithm EC \
  -pkeyopt ec_paramgen_curve:prime256v1 \
  -aes256 \
  -out private/ca.key.pem
```

> **Why EdDSA schemes (i.e. Ed25519) are not supported?**
>
> As noted in the Bugzilla ticket, certificates with Ed25519 keys are currently forbidden by the Baseline Requirements.
> All public Certificate Authorities have to adhere to the Baseline Requirements, so this key type is likely rarely used in X.509 certificates, as it can only be used for non-public certificates.
>
> ref: [stackexchange#269725](https://security.stackexchange.com/questions/269725/what-is-the-current-april-2023-browser-support-for-ed25519-certificate-signatu)

Self-sign the root CA cert:

```bash
openssl req \
  -x509 \
  -new \
  -key private/ca.key.pem \
  -sha256 \
  -days 3650 \
  -out certs/ca.crt.pem \
  -config root_ca_openssl.cnf \
  -extensions v3_ca
# Enter pass phrase for private/ca.key.pem:
# You are about to be asked to enter information that will be incorporated
# into your certificate request.
# What you are about to enter is what is called a Distinguished Name or a DN.
# There are quite a few fields but you can leave some blank
# For some fields there will be a default value,
# If you enter '.', the field will be left blank.
# -----
# Country Name (2 letter code) [AU]:CN
# State or Province Name (full name) [Some-State]:Zhejiang
# Locality Name (eg, city) []:Hangzhou
# Organization Name (eg, company) [Internet Widgits Pty Ltd]:Sweet Trust Services
# Organizational Unit Name (eg, section) []:
# Common Name (e.g. server FQDN or YOUR name) []:Banana Root CA
# Email Address []:banana@sts.azuk.top
```

Verify the root CA cert:

```bash
openssl x509 -in certs/ca.crt.pem -text -noout
#Certificate:
#    Data:
#        Version: 3 (0x2)
#        Serial Number:
#            12:ba:ee:42:43:67:d0:dc:b9:33:0e:1e:8f:02:35:18:3d:7c:a3:87
#        Signature Algorithm: ecdsa-with-SHA256
#        Issuer: C=CN, ST=Zhejiang, L=Hangzhou, O=Sweet Trust Services, CN=Banana Root CA, emailAddress=banana@sts.azuk.top
#        Validity
#            Not Before: Jul 26 10:33:04 2025 GMT
#            Not After : Jul 24 10:33:04 2035 GMT
#        Subject: C=CN, ST=Zhejiang, L=Hangzhou, O=Sweet Trust Services, CN=Banana Root CA, emailAddress=banana@sts.azuk.top
#        Subject Public Key Info:
#            Public Key Algorithm: id-ecPublicKey
#                Public-Key: (256 bit)
#                pub:
#                    04:ca:79:e0:17:c9:5b:4a:1a:41:06:60:20:dc:89:
#                    e4:21:81:aa:77:14:c3:d9:19:b5:94:84:70:98:e7:
#                    66:0f:9f:8b:8f:a5:fe:d2:c5:96:72:c0:0f:00:3c:
#                    4d:7b:4a:5e:9b:fa:79:ba:36:0c:e5:bd:d1:93:44:
#                    7c:fd:28:cf:35
#                ASN1 OID: prime256v1
#                NIST CURVE: P-256
#        X509v3 extensions:
#            X509v3 Subject Key Identifier:
#                BA:3F:EE:33:63:A0:58:C5:B7:2A:1A:B7:A9:56:5D:9B:7B:CC:19:9F
#            X509v3 Authority Key Identifier:
#                BA:3F:EE:33:63:A0:58:C5:B7:2A:1A:B7:A9:56:5D:9B:7B:CC:19:9F
#            X509v3 Basic Constraints: critical
#                CA:TRUE
#            X509v3 Key Usage: critical
#                Certificate Sign, CRL Sign
#    Signature Algorithm: ecdsa-with-SHA256
#    Signature Value:
#        30:45:02:21:00:e4:9f:1a:4b:45:16:73:0c:2c:a2:c8:78:c7:
#        e4:76:73:e3:a5:9f:83:a1:90:86:6b:a8:35:db:fa:b6:fd:4f:
#        5b:02:20:20:8f:72:2c:5a:82:fe:b4:ac:b8:db:43:44:d4:4c:
#        ea:7f:78:d1:d7:1e:3c:ae:af:48:1b:df:14:74:5f:fc:98
```

### GnuTLS certtool

Create a template config file:

```text
cn = "Banana Root CA"
organization = "Sweet Trust Services"
country = "CN"
state = "Zhejiang"
locality = "Hangzhou"
email = "banana@sts.azuk.top"
expiration_days = 3650
ca
cert_signing_key
crl_signing_key
```

Generate a private key and self-sign a certificate:

```bash
certtool --generate-privkey --key-type=ecdsa --curve=secp384r1 --outfile ca.key
#Generating a 384 bit EC/ECDSA private key (SECP384R1)...
certtool --generate-self-signed --load-privkey ca.key --template=root_ca.cfg --outfile ca.pem
#Generating a self signed certificate...
#X.509 Certificate Information:
#        Version: 3
#        Serial Number (hex): 3ec1293c1ca27cf41885c1069f1f84efdd1b0b21
#        Validity:
#                Not Before: Mon Jul 28 08:24:21 UTC 2025
#                Not After: Thu Jul 26 08:24:21 UTC 2035
#        Subject: CN=Banana Root CA,O=Sweet Trust Services,L=Hangzhou,ST=Zhejiang,C=CN
#        Subject Public Key Algorithm: EC/ECDSA
#        Algorithm Security Level: Ultra (384 bits)
#                Curve:  SECP384R1
#                X:
#                        00:d7:1e:2d:ed:29:cf:67:f1:60:01:bb:8b:06:59:36
#                        69:02:e7:f3:de:b7:ae:c4:04:dc:3d:16:f5:ff:9c:89
#                        a0:03:8c:5c:df:f8:8b:1a:37:84:72:54:2e:4c:40:9a
#                        dc
#                Y:
#                        0b:7b:84:8b:8a:9a:a8:4f:c9:17:04:52:44:72:5c:22
#                        15:00:74:7d:1e:ff:72:fb:df:f7:9f:7f:5f:be:57:96
#                        d0:be:32:2e:80:14:cd:01:10:d3:5f:f6:5f:48:1c:2c
#        Extensions:
#                Basic Constraints (critical):
#                        Certificate Authority (CA): TRUE
#                Subject Alternative Name (not critical):
#                        RFC822Name: banana@sts.azuk.top
#                Key Usage (critical):
#                        Certificate signing.
#                        CRL signing.
#                Subject Key Identifier (not critical):
#                        384632d25c02d24e7a29016f832264a21122632f
#Other Information:
#        Public Key ID:
#                sha1:384632d25c02d24e7a29016f832264a21122632f
#                sha256:112c8e7953b97dab4db15e84b764ebd26bbad3acfa427393744958b9f08244fe
#        Public Key PIN:
#                pin-sha256:ESyOeVO5fatNsV6Et2Tr0mu606z6QnOTdElYufCCRP4=
#
#Signing certificate...
```

> You're responsible for maintaining the cert serial.

## Create an intermediate CA for SSL

### OpenSSL

Still in our root ca folder, edit openssl config file.
Copy `[v3_ca]` extension to `[v3_ssl_ca]`, then modify it like:

```ini
[ v3_ssl_ca ]
# ...
basicConstraints = critical,CA:true,pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
```

- `pathlen:0`: It means this CA **only** issues certs to end-entities.
- `digitalSignature` is set here. Public intermediate CAs set it too.

The extension will be used to sign the SSL CA.
This process is done on the root CA side, but we still need openssl config file on both side.
SSL CA needs it for signing ssl certs, and generating a CSR.
However, root CA decides which extensions should be applied to the SSL CA, according to its own config file.
A CSR file is only for reference to CA operator, and the same applies to an SSL CA and its users.

Next, we create another folder named `ssl` for our SSL CA, create the directories and files.
Generate the private key, and then create a CSR for our cert:

```bash
openssl genpkey -algorithm EC \
  -pkeyopt ec_paramgen_curve:prime256v1 \
  -aes256 \
  -out private/sslca.key.pem

mkdir csr

openssl req \
  -new \
  -key private/sslca.key.pem \
  -sha256 \
  -out csr/sslca.csr.pem \
  -config ssl_ca_openssl.cnf

#Enter pass phrase for private/sslca.key.pem:
#You are about to be asked to enter information that will be incorporated
#into your certificate request.
#What you are about to enter is what is called a Distinguished Name or a DN.
#There are quite a few fields but you can leave some blank
#For some fields there will be a default value,
#If you enter '.', the field will be left blank.
#-----
#Country Name (2 letter code) [AU]:CN
#State or Province Name (full name) [Some-State]:Zhejiang
#Locality Name (eg, city) []:Hangzhou
#Organization Name (eg, company) [Internet Widgits Pty Ltd]:Sweet Trust Services
#Organizational Unit Name (eg, section) []:
#Common Name (e.g. server FQDN or YOUR name) []:Pie CA
#Email Address []:pie@sts.azuk.top
#
#Please enter the following 'extra' attributes
#to be sent with your certificate request
#A challenge password []:
#An optional company name []:
```

Sign the SSL CA with the root CA:

```bash
cd ../ca
openssl ca \
  -in ../ssl/csr/sslca.csr.pem \
  -out certs/sslca.crt.pem \
  -config root_ca_openssl.cnf \
  -extensions v3_ssl_ca \
  -days 1825
#Using configuration from root_ca_openssl.cnf
#Enter pass phrase for ./private/ca.key.pem:
#Check that the request matches the signature
#Signature ok
#Certificate Details:
#        Serial Number: 4096 (0x1000)
#        Validity
#            Not Before: Jul 26 10:48:07 2025 GMT
#            Not After : Jul 25 10:48:07 2030 GMT
#        Subject:
#            countryName               = CN
#            stateOrProvinceName       = Zhejiang
#            organizationName          = Sweet Trust Services
#            commonName                = Pie CA
#            emailAddress              = pie@sts.azuk.top
#        X509v3 extensions:
#            X509v3 Subject Key Identifier:
#                14:91:BD:8E:1C:BE:B1:03:48:6D:6B:28:42:4D:0E:F5:C4:C3:B5:77
#            X509v3 Authority Key Identifier:
#                BA:3F:EE:33:63:A0:58:C5:B7:2A:1A:B7:A9:56:5D:9B:7B:CC:19:9F
#            X509v3 Basic Constraints: critical
#                CA:TRUE, pathlen:0
#            X509v3 Key Usage: critical
#                Digital Signature, Certificate Sign, CRL Sign
#Certificate is to be certified until Jul 25 10:48:07 2030 GMT (1825 days)
#Sign the certificate? [y/n]:y
#
#
#1 out of 1 certificate requests certified, commit? [y/n]y
#Write out database with 1 new entries
#Database updated
```

And now we have our SSL CA cert.
FYI, here's the updated root ca db file:

```bash
cat serial
# 1001
cat index.txt
# V       300725104807Z           1000    unknown /C=CN/ST=Zhejiang/O=Sweet Trust Services/CN=Pie CA/emailAddress=pie@sts.azuk.top
```

### GnuTLS certtool

Create a template config file:

```text
cn = "Pie CA"
organization = "Sweet Trust Services"
country = "CN"
state = "Zhejiang"
locality = "Hangzhou"
email = "ssl@sts.azuk.top"
expiration_days = 1825
ca
cert_signing_key
crl_signing_key
# path_len=0 means this CA can only sign end-entity certificates, not other CAs.
path_len = 0
```

Generate a private key, create a csr, and sign the certificate:

```bash
certtool --generate-privkey --key-type=ecdsa --curve=secp384r1 --outfile=sslca.ke
certtool --generate-request --load-privkey sslca.key --template=ssl_ca.cfg --outfile sslca.csr
certtool --generate-certificate --load-ca-privkey ca.key --load-ca-certificate ca.pem --load-request sslca.csr --template=ssl_ca.cfg --outfile sslca.pem
#Generating a signed certificate...
#X.509 Certificate Information:
#        Version: 3
#        Serial Number (hex): 49e5139f553c2834b6c03f913d0b5870119a4a40
#        Validity:
#                Not Before: Mon Jul 28 08:35:02 UTC 2025
#                Not After: Sat Jul 27 08:35:02 UTC 2030
#        Subject: CN=Pie CA,O=Sweet Trust Services,L=Hangzhou,ST=Zhejiang,C=CN
#        Subject Public Key Algorithm: EC/ECDSA
#        Algorithm Security Level: Ultra (384 bits)
#                Curve:  SECP384R1
#                X:
#                        00:81:3c:cc:10:df:d1:77:db:a7:78:b2:f3:31:17:fc
#                        e5:16:12:9b:ab:35:98:79:68:6a:de:d9:62:a6:d4:ba
#                        ef:6e:e6:d1:cd:7f:97:42:10:e5:07:39:14:1a:df:77
#                        57
#                Y:
#                        00:82:39:25:80:63:f9:0e:ce:98:ef:f5:93:39:c0:16
#                        b7:c4:dc:0a:90:1a:7f:09:e3:c5:9f:49:dc:22:f7:3c
#                        5b:d0:a9:09:0e:bf:71:eb:21:ac:d0:d2:fb:14:80:84
#                        ec
#        Extensions:
#                Basic Constraints (critical):
#                        Certificate Authority (CA): TRUE
#                        Path Length Constraint: 0
#                Subject Alternative Name (not critical):
#                        RFC822Name: ssl@sts.azuk.top
#                Key Usage (critical):
#                        Certificate signing.
#                        CRL signing.
#                Subject Key Identifier (not critical):
#                        3201a92459b95f1a720eedba6b91198595f7af72
#                Authority Key Identifier (not critical):
#                        384632d25c02d24e7a29016f832264a21122632f
#Other Information:
#        Public Key ID:
#                sha1:3201a92459b95f1a720eedba6b91198595f7af72
#                sha256:7b0d8cd6085e2c2b368610f5a765c22949b937ffc4bd8992077fd8ad91a27389
#        Public Key PIN:
#                pin-sha256:ew2M1gheLCs2hhD1p2XCKUm5N//EvYmSB3/YrZGic4k=
#
#Signing certificate...
```

## Create an intermediate CA with Name Constraints

### GnuTLS certtool

Template file:

```ini
cn = "Pie CA"
organization = "Sweet Trust Services"
country = "CN"
state = "Zhejiang"
locality = "Hangzhou"
email = "ssl@sts.azuk.top"
expiration_days = 1825
ca
cert_signing_key
crl_signing_key
# path_len=0 means this CA can only sign end-entity certificates, not other CAs.
path_len = 0
nc_permit_dns = sts.azuk.top
nc_permit_dns = .sts.azuk.top
```

## Sign an SSL cert

### OpenSSL

For simplicity, we'll just use the `ssl` folder for our cert.

Create an OpenSSL config file for generating CSR:

> Domains issued in the cert are decided by the CA.
> The CSR is only a file we provide for their reference.

If the cert should cover multiple domains, we can use SANs for those.

```ini
[ req ]
distinguished_name      = req_distinguished_name
req_extensions = v3_req

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = AU
countryName_min                 = 2
countryName_max                 = 2
stateOrProvinceName             = State or Province Name (full name)
stateOrProvinceName_default     = Some-State
localityName                    = Locality Name (eg, city)
0.organizationName              = Organization Name (eg, company)
0.organizationName_default      = Internet Widgits Pty Ltd
organizationalUnitName          = Organizational Unit Name (eg, section)
commonName                      = Common Name (e.g. server FQDN or YOUR name)
commonName_max                  = 64
emailAddress                    = Email Address
emailAddress_max                = 64

[ req_attributes ]
challengePassword               = A challenge password
challengePassword_min           = 4
challengePassword_max           = 20
unstructuredName                = An optional company name

[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = apple.sts.azuk.top
DNS.2 = coconut.sts.azuk.top
DNS.3 = peach.sts.azuk.top
```

Generate private key and create the CSR:

Note "Common Name" should be the primary domain.

```bash
openssl genpkey \
  -algorithm EC \
  -pkeyopt ec_paramgen_curve:prime256v1 \
  -aes256 \
  -out private/server.key.pem
#Enter PEM pass phrase:
#Verifying - Enter PEM pass phrase:

openssl req \
  -new \
  -key private/server.key.pem \
  -sha256 \
  -out csr/server.csr.pem \
  -config server.cnf
#Enter pass phrase for private/server.key.pem:
#You are about to be asked to enter information that will be incorporated
#into your certificate request.
#What you are about to enter is what is called a Distinguished Name or a DN.
#There are quite a few fields but you can leave some blank
#For some fields there will be a default value,
#If you enter '.', the field will be left blank.
#-----
#Country Name (2 letter code) [AU]:CN
#State or Province Name (full name) [Some-State]:Zhejiang
#Locality Name (eg, city) []:Hangzhou
#Organization Name (eg, company) [Internet Widgits Pty Ltd]:My Services
#Organizational Unit Name (eg, section) []:
#Common Name (e.g. server FQDN or YOUR name) []:apple.sts.azuk.top
#Email Address []:apple@sts.azuk.top
```

Edit SSL CA config:

```ini
[ v3_server_ext ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @server_alt_names

[ server_alt_names ]
DNS.1 = apple.sts.azuk.top
DNS.2 = coconut.sts.azuk.top
DNS.3 = peach.sts.azuk.top
```

Sign the CSR with SSL CA:

> Note: The SSL cert validity period will gradually reduce to 47 days. [CA/Browser Forum](https://cabforum.org/working-groups/server/baseline-requirements/requirements/#632-certificate-operational-periods-and-key-pair-usage-periods)

```bash
openssl ca \
  -in ./csr/server.csr.pem \
  -out certs/server.crt.pem \
  -config ssl_ca_openssl.cnf \
  -extensions v3_server_ext \
  -days 42
#Using configuration from ssl_ca_openssl.cnf
#Enter pass phrase for ./private/sslca.key.pem:
#Check that the request matches the signature
#Signature ok
#Certificate Details:
#        Serial Number: 4096 (0x1000)
#        Validity
#            Not Before: Jul 26 13:20:23 2025 GMT
#            Not After : Sep  6 13:20:23 2025 GMT
#        Subject:
#            countryName               = CN
#            stateOrProvinceName       = Zhejiang
#            commonName                = apple.sts.azuk.top
#            emailAddress              = apple@sts.azuk.top
#        X509v3 extensions:
#            X509v3 Basic Constraints:
#                CA:FALSE
#            X509v3 Key Usage: critical
#                Digital Signature, Key Encipherment
#            X509v3 Extended Key Usage:
#                TLS Web Server Authentication
#            X509v3 Subject Alternative Name:
#                DNS:apple.sts.azuk.top, DNS:coconut.sts.azuk.top, DNS:peach.sts.azuk.top
#Certificate is to be certified until Sep  6 13:20:23 2025 GMT (42 days)
#Sign the certificate? [y/n]:y
#
#
#1 out of 1 certificate requests certified, commit? [y/n]y
#Write out database with 1 new entries
#Database updated
```

### GnuTLS certtool

Create a config:

```text
cn = "apple.sts.azuk.top"
organization = "Sweet Trust Services"
country = "CN"
state = "Zhejiang"
locality = "Hangzhou"
email = "ssl@sts.azuk.top"
expiration_days = 42
tls_www_server
dns_name = "apple.sts.azuk.top"
dns_name = "coconut.sts.azuk.top"
dns_name = "peach.sts.azuk.top"
```

Generate private key, CSR and sign the cert:

```bash
certtool --generate-privkey --key-type=ecdsa --curve=secp384r1 --outfile=server.key
#Generating a 384 bit EC/ECDSA private key (SECP384R1)...
certtool --generate-request --load-privkey server.key --template=server.cfg --outfile server.csr
#Generating a PKCS #10 certificate request...
certtool --generate-certificate --load-ca-privkey sslca.key --load-ca-certificate sslca.pem --load-request server.csr --template=server.cfg --outfile server.pem
#Generating a signed certificate...
#X.509 Certificate Information:
#        Version: 3
#        Serial Number (hex): 413ea11c552ec8ac5d6a6856bceeddc834a1bcf7
#        Validity:
#                Not Before: Mon Jul 28 13:35:37 UTC 2025
#                Not After: Mon Sep 08 13:35:37 UTC 2025
#        Subject: CN=apple.sts.azuk.top,O=Sweet Trust Services,L=Hangzhou,ST=Zhejiang,C=CN
#        Subject Public Key Algorithm: EC/ECDSA
#        Algorithm Security Level: Ultra (384 bits)
#                Curve:  SECP384R1
#                X:
#                        00:82:3d:ed:37:61:69:37:8e:ed:2d:78:d2:90:72:8e
#                        cb:48:f2:67:88:d5:91:fb:99:90:34:d2:9a:0f:65:58
#                        b3:f8:25:9d:f1:5d:8c:9f:8b:3b:28:9c:7f:5d:32:4f
#                        a9
#                Y:
#                        70:84:3d:86:da:5a:b6:ae:33:09:f8:6d:34:73:01:fd
#                        8b:ba:32:19:2e:c9:20:e6:01:93:b0:e1:98:d9:af:d1
#                        cf:9d:dd:d4:be:6c:6d:01:cd:ff:85:7e:6b:f0:36:48
#        Extensions:
#                Basic Constraints (critical):
#                        Certificate Authority (CA): FALSE
#                Subject Alternative Name (not critical):
#                        DNSname: apple.sts.azuk.top
#                        DNSname: coconut.sts.azuk.top
#                        DNSname: peach.sts.azuk.top
#                Key Purpose (not critical):
#                        TLS WWW Server.
#                Key Usage (critical):
#                        Digital signature.
#                Subject Key Identifier (not critical):
#                        c06ad285dcb5c4cc2e7b2b646e871c23a714e9a5
#                Authority Key Identifier (not critical):
#                        3201a92459b95f1a720eedba6b91198595f7af72
#Other Information:
#        Public Key ID:
#                sha1:c06ad285dcb5c4cc2e7b2b646e871c23a714e9a5
#                sha256:992e6cb93d7f5d5e5f8c686ab608ca8d68142ace2f0bbec885328a9cf909a704
#        Public Key PIN:
#                pin-sha256:mS5suT1/XV5fjGhqtgjKjWgUKs4vC77IhTKKnPkJpwQ=
#
#Signing certificate...
```

## Run a test HTTP server

### Preparation

#### Decrypt your private key

OpenSSL:

```bash
# EC key
openssl ec -in key.pem -out decrypted.pem
# RSA key
openssl rsa -in key.pem -out decrypted.pem
```

#### Bundle certs into a fullchain pem

```bash
cat server.pem > fullchain.pem
cat ssl_ca.pem >> fullchain.pem
cat root_ca.pem >> fullchain.pem
```

### Caddy

#### Manually

```caddyfile
# caddy's automatic https listens to 80 and 443
# disable them for test purpose
{
  auto_https off
}

apple.sts.azuk.top:32123 {
  tls ./fullchain.pem ./server.key
  respond "Hello"
}

coconut.sts.azuk.top:32123 {
  tls ./fullchain.pem ./server.key
  respond "Hello"
}
```

```bash
curl -v --cacert ca.pem https://apple.sts.azuk.top:32123
```

#### Mutual TLS (client certificate verification)


#### Acme


